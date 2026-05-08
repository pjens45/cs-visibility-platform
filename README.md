# CS Command Center

A real-time customer support dashboard built on Google Apps Script.

## The problem

Our CS tech stack is composed of a curated selection of purpose-built tools — Zendesk for tickets, Aircall for phones, Nicereply for CSAT, Meta Business Suite for social. From a management perspective this creates a fragmented view of operations. There's no single place to assess queue health, SLA compliance, agent workload, and customer satisfaction in real time. No way to deploy resources efficiently or follow up on key events promptly without checking four different apps.

## The solution

A single Google Apps Script file that pulls from all four APIs and renders a consolidated real-time command center inside Google Sheets. Refreshes every 5 minutes. Zero additional cost, no new tools to learn, no vendor to manage.

### What it tracks

**Email (Zendesk)**
- Open ticket count, on-hold count, unassigned count
- SLA compliance — tickets waiting longer than configurable business hours without a first reply
- Per-agent breakdown: assigned, past SLA, longest wait, solved today
- New SAS answering service tickets (detection via tag, subject, and requester)
- Oldest waiting tickets table with clickable links to Zendesk
- Flagged tickets (high priority, warranty, escalated, VIP)
- Email CSAT survey results from Nicereply (last 24 hours)

**Phone (Aircall)**
- Answer rate (business hours only, Mon–Fri 6a–5p PST)
- Per-agent inbound/outbound call volume with short-call breakdown
- Missed calls forwarded to answering service with reason detection
- SMS activity per agent with recent message preview
- Post-call CSAT survey results

**Social (Meta Business Suite)**
- Facebook Messenger DMs — unread count, conversation list with excerpts
- Instagram DMs — merged with Messenger, tagged by platform (FB/IG)
- Facebook and Instagram post comments (last 24 hours)
- Instagram @mentions and tags (last 24 hours)
- Deep links to Meta Business Suite inbox for each conversation and comment
- Token expiry monitoring with 7-day warning

**Customer satisfaction (Nicereply)**
- Email CSAT score and percentage (last 24 hours)
- Individual survey responses with score, customer name, ticket link, and timestamp

**Dashboard health indicators**
- Email status: Healthy / Watch / At Risk (based on SLA breach count)
- Phone status: Healthy / Watch / At Risk (based on answer rate)
- All thresholds configurable via Script Properties

## Tech stack

- **Runtime**: Google Apps Script (V8)
- **UI**: Google Sheets (formatted programmatically — no add-ons)
- **APIs**: Zendesk REST API, Aircall REST API, Nicereply REST API, Meta Graph API v25.0
- **Auth**: API tokens stored in Apps Script Script Properties (never committed to code)
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
| `META_PAGE_TOKEN` | `your_meta_page_access_token` |
| `META_PAGE_ID` | `your_facebook_page_id` |

5. Select `initializeSheet` from the function dropdown, click **Run**
6. Select `setupTrigger` from the function dropdown, click **Run**

### Optional Script Properties

SLA thresholds default to sensible values but can be overridden per-deployment:

| Property | Default | Description |
|----------|---------|-------------|
| `SLA_EMAIL_GREEN` | `12` | Business hours before a ticket is "past SLA" |
| `SLA_EMAIL_YELLOW` | `24` | Threshold for yellow/watch state |
| `SLA_BACKLOG_GREEN` | `30` | Open ticket count — green ceiling |
| `SLA_BACKLOG_YELLOW` | `50` | Open ticket count — yellow ceiling |
| `SLA_PHONE_GREEN` | `75` | Answer rate % — green floor |
| `SLA_PHONE_YELLOW` | `60` | Answer rate % — yellow floor |
| `SLA_FRT_GREEN` | `12` | Median first reply time (biz hrs) — green ceiling |
| `SLA_FRT_YELLOW` | `24` | Median first reply time (biz hrs) — yellow ceiling |
| `SLA_WAIT_GREEN` | `30` | Average wait time (min) — green ceiling |
| `SLA_WAIT_YELLOW` | `60` | Average wait time (min) — yellow ceiling |

### Meta Business Suite setup

1. Create a Meta App at [developers.facebook.com](https://developers.facebook.com)
2. Add the Messenger product
3. Generate a Page Access Token with permissions: `pages_messaging`, `pages_read_engagement`, `instagram_basic`, `instagram_manage_messages`, `instagram_manage_comments`, `business_management`
4. Extend to a long-lived token (60-day expiry) via the Access Token Debugger
5. The script auto-discovers your Instagram Business Account ID from the linked Page

## Configuration

All configuration is in the `CONFIG` object at the top of the script:

- `agents` — CS team member names (drives per-agent breakdowns)
- `excludeAgents` — people who use Zendesk but aren't on the CS team (filtered from all stats)
- `excludeSMSLines` — Aircall lines to exclude from SMS tracking (sales reps, etc.)
- `supportNumbers` — Aircall phone numbers for CS support lines
- `answeringServiceNumber` — external answering service number (for SAS forwarding detection)
- `businessHours` — timezone, start/end hour, work days (for answer rate and SLA calculations)

## Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Zendesk   │  │   Aircall   │  │  Nicereply  │  │  Meta Graph │
│  REST API   │  │  REST API   │  │  REST API   │  │  API v25.0  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       └────────┬───────┴────────────────┴────────────────┘
                │
      ┌─────────▼──────────┐
      │  Google Apps Script  │
      │  (Code.gs — single   │
      │   file, ~3500 LOC)   │
      └─────────┬──────────┘
                │
      ┌─────────▼──────────┐
      │   Google Sheets      │
      │   Dashboard tab      │
      │   (auto-formatted)   │
      └────────────────────┘
```

The script runs on a 5-minute trigger. Each refresh fetches fresh data from all four APIs, processes it, and overwrites the Dashboard sheet. A staging sheet is used during writes to prevent the dashboard from flickering mid-update. Raw data is also written to hidden tabs for debugging.

## Security

All API credentials and SLA thresholds are stored in Google Apps Script Script Properties — never in the source code. The committed file contains no secrets.
