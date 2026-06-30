# AI Meeting Notes Processor — n8n + Groq AI

![n8n](https://img.shields.io/badge/n8n-workflow-orange?logo=n8n)
![Groq AI](https://img.shields.io/badge/Groq-llama--3.3--70b-blue)
![Gmail](https://img.shields.io/badge/Gmail-integration-red?logo=gmail)
![Google Sheets](https://img.shields.io/badge/Google-Sheets-green?logo=googlesheets)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> Send a meeting transcript — get back a structured summary, action items, decisions, and next steps emailed to all attendees in seconds. Works with Otter.ai, Fireflies, Zoom, or any transcription tool.

---

## The Problem

After every meeting, someone has to:
- Re-read the transcript to write a summary
- Identify and assign action items
- Note decisions made
- Write up next steps
- Email everything to attendees

This takes 15–30 minutes per meeting. For a team with 5 meetings a week, that's 75–150 minutes of admin every week.

## The Solution

Send the transcript to a webhook — this workflow uses Groq AI to extract everything in ~3 seconds, formats it into a beautiful HTML email, sends it to all attendees, and logs the meeting to Google Sheets. **What used to take 20 minutes now takes 5 seconds.**

---

## Architecture

```
Meeting Ends → Transcript Ready
          │
          ▼
  [Webhook Trigger]
  {transcript, attendees, title}
          │
          ▼
[Groq AI: Extract structured data]
  → summary, action_items,
    decisions, next_steps,
    key_topics (JSON)
          │
          ▼
[Code Node: Format HTML email
  with branded design + tables]
          │
    ┌─────┴─────┐
    ▼           ▼
[Gmail:      [Sheets:
 Send to      Log meeting
 Attendees]   metadata]
          │
          ▼
 [Respond to Webhook]
```

### Nodes

| Node | Purpose |
|------|---------|
| **Webhook** | Receives transcript + meeting metadata |
| **HTTP / Groq AI** | Extracts structured notes as JSON via `llama-3.3-70b` |
| **Code** | Transforms JSON into professional HTML email |
| **Gmail** | Sends formatted notes to all attendees |
| **Google Sheets** | Logs meeting to searchable history |
| **Respond to Webhook** | Returns confirmation with counts |

---

## What Groq AI Extracts

From the transcript, the AI automatically identifies:

- **Summary** — 2-3 sentence executive overview of the meeting
- **Action Items** — each with owner, task description, and due date
- **Decisions** — key choices made during the meeting
- **Next Steps** — immediate follow-up actions
- **Key Topics** — main themes discussed

All returned as structured JSON, then formatted into a branded email.

---

## Demo Payload

```json
POST /webhook/process-meeting
Content-Type: application/json

{
  "meeting_title": "Q1 Product Roadmap Review",
  "meeting_date": "2025-01-16",
  "attendees": "Sarah, Marcus, Priya, Tom",
  "attendees_emails": "sarah@co.com,marcus@co.com,priya@co.com",
  "transcript": "Sarah: Good morning everyone. Let's start with the roadmap review for Q1. Marcus, can you give us an update on the API integration? Marcus: Sure, we're at 70% completion. We hit a blocker with the authentication layer but Priya has a fix ready. We should be done by Friday. Priya: That's right. I'll push the auth fix today and we should be fully integrated by end of week. Sarah: Great. Decision: we're moving the API launch date to January 24th. Tom, can you update the marketing calendar? Tom: Will do. I'll also need the API documentation by Monday to brief the sales team. Marcus: I'll have docs ready by Sunday. Sarah: Perfect. Next steps: Priya ships auth fix today, Marcus delivers docs Sunday, Tom updates calendar by EOD. We'll reconvene Thursday for a final check. Sound good? All: Agreed."
}
```

**Email received by attendees:**

```
📋 Meeting Notes: Q1 Product Roadmap Review — 2025-01-16

Executive Summary
The team reviewed the Q1 roadmap with focus on the API integration, 
currently at 70% completion...

✅ Action Items
┌─────────┬──────────────────────────────┬──────────┐
│ Owner   │ Task                         │ Due Date │
├─────────┼──────────────────────────────┼──────────┤
│ Priya   │ Ship authentication fix      │ Today    │
│ Marcus  │ Deliver API documentation    │ Sunday   │
│ Tom     │ Update marketing calendar    │ EOD      │
└─────────┴──────────────────────────────┴──────────┘

🏆 Decisions Made
• API launch date moved to January 24th

🚀 Next Steps
• Final check meeting scheduled for Thursday
• Sales team briefing after docs delivered
```

---

## Setup Guide

### Prerequisites

- n8n instance
- Google account (Gmail + Sheets)
- Groq API key (free at [console.groq.com](https://console.groq.com))
- Any transcription tool: Otter.ai, Fireflies, Zoom AI, Rev, or manual transcripts

### Step 1 — Import the workflow

n8n → **Workflows** → **Import from File** → `workflow.json`

### Step 2 — Configure credentials

| Credential | Type | Setup |
|-----------|------|-------|
| Groq API | HTTP Header Auth | Header: `Authorization` · Value: `Bearer gsk_...` |
| Gmail | OAuth2 | Connect Google account |
| Google Sheets | OAuth2 | Same Google account |

### Step 3 — Configure Google Sheets

Create a sheet named **Meeting Log** with columns:

| Date | Meeting Title | Attendees | Summary | Action Items Count | Decisions Count | Key Topics | Processed At |

Replace `YOUR_GOOGLE_SHEET_ID` in the Sheets node.

### Step 4 — Activate & get webhook URL

1. Click **Activate**
2. Copy the webhook URL from the Webhook node

### Step 5 — Connect your transcription tool

**Option A — Otter.ai / Fireflies webhook**
- Set the workflow's webhook URL as the post-meeting webhook in your transcription tool's settings

**Option B — Zapier / Make middleware**
- Use Zapier to watch for new Otter.ai transcripts → POST to this webhook

**Option C — Manual**
```bash
curl -X POST https://your-n8n.domain/webhook/process-meeting \
  -H "Content-Type: application/json" \
  -d '{
    "meeting_title": "Sprint Planning",
    "meeting_date": "2025-01-16",
    "attendees": "Alice, Bob, Carol",
    "attendees_emails": "alice@co.com,bob@co.com",
    "transcript": "YOUR FULL TRANSCRIPT HERE"
  }'
```

---

## Customization Ideas

- **Notion integration** — save meeting notes to a Notion database instead of/alongside Sheets
- **Slack notification** — post summary to team Slack channel
- **Calendar event creation** — auto-create follow-up meeting in Google Calendar
- **Jira/Linear tickets** — auto-create tickets for action items with assigned owners
- **Language support** — the Groq AI handles transcripts in any language
- **Confidence scoring** — add prompt engineering to flag uncertain extractions

---

## Transcription Tool Integrations

| Tool | Integration Method |
|------|--------------------|
| **Fireflies.ai** | Native webhook support → POST to n8n URL |
| **Otter.ai** | Export → Zapier → n8n |
| **Zoom AI Summary** | Zoom webhook → n8n |
| **Rev** | Email notification → Gmail trigger → n8n |
| **Manual / Teams** | Copy-paste transcript into API call |

---

## Tech Stack

- **n8n** — workflow automation
- **Groq API** (`llama-3.3-70b-versatile`) — AI transcript analysis with JSON mode
- **Gmail** — HTML email delivery to attendees
- **Google Sheets** — searchable meeting history

---

## Performance

- **AI processing time:** ~2-4 seconds (Groq is ultra-fast)
- **Total workflow time:** ~6-8 seconds from webhook to sent email
- **Transcript length:** handles up to ~50,000 tokens (full-day meeting)
- **Languages:** any language Llama 3.3 supports (30+)

---

## Author

Built by **Dilovar Sam** — AI automation specialist.

[GitHub](https://github.com/dsam-ai) · [Portfolio](https://github.com/dsam-ai?tab=repositories)

---

## License

MIT — free to use and adapt.
