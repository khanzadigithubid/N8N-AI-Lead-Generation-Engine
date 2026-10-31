# 🚀 N8N AI Lead Generation Engine

> A fully automated, AI-powered B2B lead generation and outreach system built on [n8n](https://n8n.io). It sources prospects from Apollo, scores them with GPT-4o-mini, sends personalized cold email sequences via Gmail, classifies replies, syncs hot leads to HubSpot, and posts daily reports to Slack — all on autopilot.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Workflow Flows](#workflow-flows)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
- [Configuration](#configuration)
- [Google Sheets Schema](#google-sheets-schema)
- [How It Works](#how-it-works)
- [Customization](#customization)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This n8n workflow automates the entire B2B outbound sales pipeline — from finding the right prospects to booking calls — without manual intervention.

**What it does every weekday:**

| Time | Action |
|------|--------|
| 8:00 AM | Sources new leads from Apollo, enriches them, and scores with AI |
| 10:00 AM | Sends cold emails and follow-ups to qualified leads |
| Every minute | Monitors inbox, classifies replies, and routes hot leads to HubSpot |
| 6:00 PM | Posts a daily performance report to Slack |

**Key numbers (default config):**
- Up to **25 new leads enriched** per day (1 Apollo credit each)
- **3-step email sequence**: intro → follow-up (day 3) → break-up (day 7)
- Leads scoring **60+** out of 100 are queued for outreach
- Interested replies get an **automatic booking link** sent back

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        FLOW 1 — SOURCING                        │
│  Trigger (8AM / Manual)                                         │
│    → ICP Config → Get Existing Leads → Apollo Search            │
│    → Filter Duplicates → Apollo Enrich (1 credit/lead)          │
│    → Fetch Website → AI Score (GPT-4o-mini)                     │
│    → Save to Google Sheets (status: new / disqualified)         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                       FLOW 2 — OUTREACH                         │
│  Trigger (10AM / Manual)                                        │
│    → Get All Leads → Select Due Leads (max 25/day)              │
│    → Loop: AI Write Email → Gmail Send / Reply                  │
│    → Update Sheet → Wait 40–90s → Next Lead                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    FLOW 3 — REPLY HANDLING                      │
│  Gmail Trigger (every minute)                                   │
│    → Find Lead by Thread ID → AI Classify Reply                 │
│    → Update Lead Status                                         │
│    → Hot Lead? → Slack Alert + HubSpot Upsert                   │
│               → Interested? → Auto-send Booking Link            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     FLOW 4 — DAILY REPORT                       │
│  Trigger (6PM Mon-Fri)                                          │
│    → Get All Leads → Build Stats → Post to Slack                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workflow Flows

### Flow 1 — Lead Sourcing & AI Scoring
Runs at **8:00 AM Mon–Fri** (or manually).

1. Reads your **ICP Config** (titles, locations, company size, description)
2. Loads existing leads from Google Sheets to avoid duplicates
3. Calls **Apollo People Search** to find matching prospects
4. Filters out already-seen contacts
5. **Enriches each person** via Apollo (reveals verified work email)
6. Scrapes the prospect's company website (up to 3,000 chars)
7. Sends everything to **GPT-4o-mini** which returns:
   - Score (0–100), Tier (A/B/C)
   - One-sentence fit reason
   - Likely pain point
   - Personalization hook (a real detail from the website)
8. Saves to Google Sheets as `new` (score ≥ min_score + verified email) or `disqualified`

### Flow 2 — 3-Step Email Outreach
Runs at **10:00 AM Mon–Fri** (or manually).

| Step | Trigger | Email Type | Max Length |
|------|---------|-----------|-----------|
| 1 | status = `new` | Cold intro with personalization hook | 90 words |
| 2 | 3 days after step 1, no reply | Follow-up (same thread) | 50 words |
| 3 | 4 days after step 2, no reply | Break-up email | 40 words |

- Capped at **25 emails/day**, best-scoring leads first
- Random **40–90 second delay** between sends (avoids spam filters)
- GPT-4o-mini writes every email from scratch using the lead's pain point and hook

### Flow 3 — Reply Handling & CRM
Runs **every minute** via Gmail trigger.

- Matches incoming emails to leads using Gmail **thread ID**
- GPT-4o-mini classifies the reply into one of:
  - `interested` — wants a call/demo/pricing
  - `question` — asked something but hasn't committed
  - `not_interested` — polite decline
  - `unsubscribe` — asked to stop
  - `out_of_office` — auto-reply
- For **interested** or **question**: fires Slack alert + upserts contact to HubSpot
- For **interested**: auto-replies with your calendar booking link
- Lead is never emailed again once they reply

### Flow 4 — Daily Slack Report
Runs at **6:00 PM Mon–Fri**.

Posts a summary including:
- Leads sourced and qualified today
- Emails sent today
- Replies received and hot leads
- Overall reply rate
- Tier A leads waiting in queue
- Total database size

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| [n8n](https://n8n.io) | Workflow automation engine |
| [Apollo.io](https://apollo.io) | Prospect search & email enrichment |
| [OpenAI GPT-4o-mini](https://openai.com) | Lead scoring, email writing, reply classification |
| [Google Sheets](https://sheets.google.com) | Lead database |
| [Gmail](https://gmail.com) | Email sending and inbox monitoring |
| [HubSpot](https://hubspot.com) | CRM — stores hot leads as contacts |
| [Slack](https://slack.com) | Hot lead alerts and daily reports |

---

## Prerequisites

- **n8n** instance (self-hosted or [n8n Cloud](https://n8n.io/cloud))
- **Apollo.io** account (free tier works; enrichment costs 1 credit/lead)
- **OpenAI** API key (GPT-4o-mini is very cost-efficient)
- **Google** account with Sheets and Gmail enabled
- **HubSpot** account (free CRM tier is sufficient)
- **Slack** workspace with a channel for alerts

---

## Setup Guide

### Step 1 — Create the Google Sheet

Create a new Google Sheet and name the first tab **`Leads`**. Add these column headers in row 1:

```
apollo_id | name | first_name | email | title | company | website | linkedin_url
industry | employees | country | score | tier | score_reason | pain_point
personalization_hook | status | step | added_at | sent_at | thread_id
message_id | subject | reply_summary | replied_at
```

Copy the Sheet ID from the URL:
`https://docs.google.com/spreadsheets/d/`**`YOUR_SHEET_ID`**`/edit`

### Step 2 — Import the Workflow

1. Open your n8n instance
2. Go to **Workflows → Import from file**
3. Upload `N8N-AI-Lead-Generation-Engine.json`

### Step 3 — Set Up Credentials

Create the following credentials in n8n (**Settings → Credentials**):

| Credential Name | Type | Details |
|----------------|------|---------|
| Apollo API Key | HTTP Header Auth | Header name: `x-api-key`, Value: your Apollo API key |
| Google Sheets account | Google Sheets OAuth2 | Authorize via Google |
| OpenAI account | OpenAI API | Your OpenAI API key |
| Gmail account | Gmail OAuth2 | Authorize via Google |
| Slack account | Slack API | Bot token with `chat:write` scope |
| HubSpot account | HubSpot App Token | Private app token with contacts write scope |

### Step 4 — Configure the Workflow

Open the workflow and update these nodes:

**`ICP Config` node** — edit the JS to match your target market:
```js
person_titles: ['CEO', 'Founder', 'Head of Operations'],
person_locations: ['United States', 'United Kingdom'],
organization_num_employees_ranges: ['11,50', '51,200'],
daily_target: 25,   // Apollo credits used per day
min_score: 60,      // minimum AI score to qualify
icp_description: 'Describe your ideal customer here...'
```

**All Google Sheets nodes** — replace `YOUR_GOOGLE_SHEET_ID` with your actual Sheet ID.

**`AI: Write Email` node** — replace placeholders in the system prompt:
```
YOUR_NAME        → your first name
YOUR_COMPANY     → your company name
```

**`Gmail: Send Booking Link` node** — replace:
```
YOUR_BOOKING_LINK  → your Calendly / cal.com link
YOUR_NAME          → your first name
```

**Slack nodes** — replace `YOUR_SLACK_CHANNEL_ID` with your channel ID.

### Step 5 — Assign Credentials to Nodes

In each node that shows a credential dropdown, select the credential you created in Step 3.

### Step 6 — Test

1. Set `daily_target: 3` in ICP Config
2. Run **Flow 1** manually using the "Manual: Run Sourcing" trigger
3. Verify 3 leads appear in your Google Sheet
4. Run **Flow 2** manually using the "Manual: Run Outreach" trigger
5. Check Gmail sent folder
6. Increase `daily_target` when satisfied

---

## Configuration

All tunable parameters live in the **`ICP Config`** node:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `person_titles` | Founder, CEO, COO… | Job titles to target |
| `person_locations` | US, UK, CA, AU | Countries to search |
| `organization_num_employees_ranges` | 11–200 | Company size range |
| `daily_target` | 25 | Max Apollo enrichments per day |
| `min_score` | 60 | Minimum AI score to qualify a lead |
| `icp_description` | (see node) | Freetext description fed to GPT for scoring |

Outreach timing (in **`Select Due Leads`** node):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DAILY_CAP` | 25 | Max outreach emails per day |
| `GAP_1` | 3 days | Wait before follow-up email |
| `GAP_2` | 4 days | Wait before break-up email |

---

## Google Sheets Schema

| Column | Description |
|--------|-------------|
| `apollo_id` | Unique Apollo person ID (deduplication key) |
| `name` / `first_name` | Contact name |
| `email` / `email_status` | Work email and Apollo verification status |
| `title` / `company` | Job title and company name |
| `website` / `linkedin_url` | Company website and LinkedIn profile |
| `industry` / `employees` / `country` | Firmographic data |
| `score` | AI fit score (0–100) |
| `tier` | A (80+), B (60–79), C (below 60) |
| `score_reason` | One-sentence AI explanation |
| `pain_point` | Likely automation pain point |
| `personalization_hook` | Specific website detail for email opener |
| `status` | `new` → `contacted` → `interested` / `not_interested` / `sequence_complete` |
| `step` | Current step in sequence (0–3) |
| `added_at` | Timestamp when lead was sourced |
| `sent_at` | Timestamp of last email sent |
| `thread_id` / `message_id` | Gmail thread for reply matching |
| `subject` | Email subject line |
| `reply_summary` | AI one-sentence summary of their reply |
| `replied_at` | Timestamp of reply |

---

## How It Works

```
Apollo Search (free)
      │
      ▼
Filter duplicates using Sheet
      │
      ▼
Apollo Enrich → gets verified work email  (1 credit)
      │
      ▼
Scrape company website (3,000 chars)
      │
      ▼
GPT-4o-mini scores lead → score, tier, pain point, hook
      │
   score ≥ 60 + verified email?
      │
   YES → status = "new"       NO → status = "disqualified"
      │
      ▼ (next morning)
GPT-4o-mini writes personalized cold email using hook + pain point
      │
      ▼
Gmail sends email → sheet updated → wait 40–90s → next lead
      │
      ▼ (reply detected)
GPT-4o-mini classifies reply
      │
   interested / question?
      │
   YES → Slack alert + HubSpot contact + auto booking link reply
```

---

## Customization

**Change the AI model** — open any `OpenAI Model` node and switch to `gpt-4o` for higher quality or `gpt-3.5-turbo` for lower cost.

**Add more email steps** — edit the `Select Due Leads` node to add `GAP_3` and `next_step = 4`, then update the `AI: Write Email` prompt template.

**Target a different persona** — update `person_titles`, `person_locations`, and `icp_description` in `ICP Config`.

**Use a different CRM** — replace the HubSpot node with Salesforce, Pipedrive, or any CRM supported by n8n.

**Add SMS or WhatsApp** — fork the reply-handling flow to send a Twilio SMS on `interested` replies.

---

## Contributing

Pull requests are welcome. For major changes please open an issue first to discuss what you'd like to change.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes (`git commit -m 'Add my feature'`)
4. Push to the branch (`git push origin feature/my-feature`)
5. Open a Pull Request

---

## License

This project is licensed under the [MIT License](LICENSE).

---

<div align="center">

Built with ❤️ using [n8n](https://n8n.io) · [Apollo.io](https://apollo.io) · [OpenAI](https://openai.com)

</div>
