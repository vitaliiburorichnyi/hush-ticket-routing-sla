# HUSH - AI Ticket Routing System with SLA Escalation

Automated ticket classification, prioritization, and SLA escalation for a B2C e-commerce support team.

Built for HUSH, a premium bag store receiving 50-80 support tickets per day across Instagram, Telegram, Facebook, website form, and email.

---

## What This Does

**Track 1 - Ticket Routing**

Every incoming email is automatically:
- Classified into one of 5 categories using AI with Chain of Thought reasoning
- Assigned a priority (P1-P4) and SLA deadline based on working hours
- Logged to Google Sheets with 15 fields including reasoning and sentiment
- Notified to the right owner in Slack within seconds

**Track 2 - SLA Watcher**

Every 5 minutes the system:
- Checks all open tickets for SLA breach (working hours only, 09:00-22:00)
- Computes how many minutes overdue each ticket is
- Fires a 3-level escalation chain based on overdue time

---

## Ticket Categories

| Category | Description | Priority |
|---|---|---|
| `product-info` | Size, material, availability questions - client deciding to buy now | P1 - 10 min |
| `payment-billing` | Failed payment, double charge, blocked money, refund | P1 - 10 min |
| `complaint-legal` | Legal threat, damaged item, aggressive complaint | P2 - 30 min |
| `order-shipping-returns` | Package tracking, returns, exchanges, delivery delays | P3 - 2 hours |
| `influencer` | Gifting, collab, PR partnership requests | P4 - 24 hours |
| `spam` | Advertising, irrelevant messages | Filtered out |

---

## Escalation Chain

| Level | Trigger | Recipient | Channel |
|---|---|---|---|
| L0 | 0-30 min overdue | On-duty operator | Slack DM |
| L1 | 30-60 min overdue | Whole team | #cx-escalations Slack channel |
| L2 | 60+ min overdue | Owner | Email with full ticket context |

---

## Working Hours Logic

SLA timers run during working hours only (09:00-22:00).

- Ticket arrives at 22:30 - SLA deadline starts at 09:00 next day plus SLA minutes
- Ticket arrives at 07:00 - SLA deadline starts at 09:00 same day plus SLA minutes
- Ticket arrives at 14:00 - normal calculation

SLA Watcher also stops entirely outside working hours. No false escalation alerts overnight.

---

## Tech Stack

| Tool | Role |
|---|---|
| n8n (self-hosted) | Workflow automation, both tracks |
| OpenAI GPT-4o-mini | Ticket classification with Chain of Thought |
| Gmail API | Ticket ingestion trigger + L2 escalation email |
| Google Sheets | Ticket log, SLA tracking, status management |
| Slack | New ticket notifications + escalation alerts |

---

## Repository Structure

```
/
├── 01_Ticket_Routing.json      # Track 1 - Gmail trigger, AI classifier, Sheets log, Slack notify
├── 02_SLA_Watcher.json         # Track 2 - cron trigger, breach detection, escalation chain
└── README.md
```

---

## Setup

### 1. Google Sheets

Create a spreadsheet called `Customer Care - Tickets`. Rename Sheet1 to `Tickets`.

Add these headers in row 1 in this exact order:

```
ticket_id | created_at | from_email | subject | category | priority | owner |
sla_minutes | sla_deadline | status | summary | reasoning | sentiment |
requires_human | thread_link
```

Copy the document ID from the URL and replace `YOUR_GOOGLE_SHEETS_DOC_ID` in both workflow files.

### 2. Slack

Create two channels:
- `#cx-tickets` - new ticket notifications
- `#cx-escalations` - L1 escalation alerts

Create a Slack App at https://api.slack.com/apps with these bot token scopes:
```
chat:write
chat:write.public
channels:read
im:write
users:read
```

Install the app and invite it to both channels with `/invite @your-app-name`.

Replace the following placeholders in the workflow files:
- `YOUR_CX_TICKETS_CHANNEL_ID` - right-click channel, View details, copy Channel ID (C0XXXXXXXXX)
- `YOUR_CX_ESCALATIONS_CHANNEL_ID` - same for escalations channel
- `YOUR_OWNER_SLACK_MEMBER_ID` - click your profile, More, Copy Member ID (U0XXXXXXXXX)

### 3. Credentials in n8n

After importing the workflows, connect credentials for each node:

| Placeholder | Credential type |
|---|---|
| `YOUR_GMAIL_CREDENTIAL_ID` | Gmail OAuth2 |
| `YOUR_OPENAI_CREDENTIAL_ID` | OpenAI API key |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Google Sheets OAuth2 |
| `YOUR_SLACK_CREDENTIAL_ID` | Slack OAuth2 (Bot Token xoxb-...) |

Replace `YOUR_OWNER_EMAIL` in the `Email: Owner (L2)` node with the owner's email address.

### 4. Import and Activate

1. n8n - Workflows - Import from File - select `01_Ticket_Routing.json`
2. Repeat for `02_SLA_Watcher.json`
3. Connect all credentials and replace all placeholders
4. Toggle Active on both workflows

---

## How to Close a Ticket

When a client request is resolved, open Google Sheets and change the `status` column from `open` to `closed`. The SLA Watcher reads only `status = open` rows, so closed tickets drop out of the escalation scope automatically.

---

## Key Technical Notes

**Parse AI Output node**
The Agent node returns AI response as a plain string, not a parsed object. A Code node between AI Classifier and Flatten Result handles the conversion:
```javascript
const raw = $input.item.json.output;
const parsed = JSON.parse(raw);
return [{ json: { output: parsed } }];
```

**escalation_level calculation**
Both `minutes_overdue` and `escalation_level` are computed in the same Set node. Because n8n evaluates fields in the same node simultaneously, `escalation_level` cannot reference `$json.minutes_overdue` - it must repeat the full calculation inline.

**DateTime parsing**
`DateTime.fromFormat()` fails on ISO strings returned by Google Sheets due to millisecond format inconsistencies. The fix uses native JavaScript: `new Date($json.sla_deadline).getTime()`.

---

## AI Classifier Prompt

The classifier uses a 5-block prompt structure:

1. **Role** - senior support coordinator for HUSH
2. **Context** - 5 categories with urgency logic and business rationale
3. **Instructions** - priority matrix P1-P4 with SLA times and owner assignments
4. **Chain of Thought** - 3 mandatory reasoning dimensions before assigning priority: urgency signals, business impact, client context
5. **Output format** - strict JSON with 8 fields, no markdown wrapper

Special override rules embedded in the prompt:
- Words "court", "lawyer", "press", "attorney" always trigger complaint-legal P2
- Gifting and collab requests are always influencer P4, never P1 regardless of urgency language
- Explicit request to speak with a human sets `requires_human: true`
- Language detected from message body, not sender name

---

## Test Results

10 test tickets processed with 0 misclassifications:

| Ticket | Expected | Result |
|---|---|---|
| Paid twice for order | payment-billing P1 | payment-billing P1 |
| Does bag fit 13 inch laptop | product-info P1 | product-info P1 |
| Stock availability (Ukrainian) | product-info P1 | product-info P1 |
| Order not arrived | order-shipping-returns P3 | order-shipping-returns P3 |
| Collaboration proposal | influencer P4 | influencer P4 |
| URGENT partnership deadline | influencer P4 | influencer P4 (special rule held) |
| Exchange request | order-shipping-returns P3 | order-shipping-returns P3 |
| Damaged item legal threat | complaint-legal P2 | complaint-legal P2 |
| Return request | order-shipping-returns P3 | order-shipping-returns P3 |
| SEO spam | spam | filtered, not logged |

---

## Known Limitations

| Issue | Description | Impact |
|---|---|---|
| Manual ticket close | Operator must change `status` to `closed` in Sheets manually after resolving | If forgotten, ticket keeps triggering escalations every 5 min |
| Repeat escalation alerts | No flag to track if L2 email was already sent - breach at L2 sends owner an email every 5 min until closed | Owner inbox gets flooded on long-unresolved tickets |
| No duplicate guard | If the same email is processed twice (e.g. trigger runs twice quickly), a duplicate row is created in Sheets | Inflated ticket counts, duplicate Slack notifications |
| Email only in prototype | Phase 1 covers Gmail only. Instagram Direct, Telegram, Facebook, and website form are not connected yet | All tickets must come via email for automation to work |
| No retry on AI failure | If the AI Classifier or Parse AI Output node fails, the ticket is silently lost - no error notification | Tickets can be missed without the operator knowing |

---

## Next Steps

**Priority fixes (before production use):**

1. **Add escalation_sent flag to Sheets**
   Add a column `escalation_sent` (values: none / L0 / L1 / L2). Before firing any escalation, check this column. Only escalate if the current level is higher than what was already sent. Prevents repeat L2 emails on the same ticket.

2. **Add Slack close button**
   Add a Slack button to the notification message. When the operator clicks it, a webhook triggers an n8n workflow that updates the Sheets row `status` to `closed` automatically. Removes the manual Sheets step entirely.

3. **Add duplicate guard**
   Add Gmail label `Ticketed` after processing. Update the Gmail Trigger filter to `is:unread newer_than:1d -label:Ticketed`. Prevents the same email from being processed twice.

4. **Add error notification node**
   After Parse AI Output, add an error handler. If JSON.parse fails (AI returned malformed output), send a Slack alert with the raw email subject so the operator can handle it manually.

**Phase 2 - additional channels:**

5. **Instagram Direct**
   Connect via Instagram Messaging webhook. Each DM feeds into the same AI Classifier. Requires a Facebook App with instagram_manage_messages permission.

6. **Telegram**
   Connect via Telegram Bot webhook. Simple to add - one Webhook node replaces the Gmail Trigger for that channel, rest of the pipeline stays identical.

7. **Facebook Business**
   Connect via Facebook Messenger webhook. Similar to Instagram setup, shares the same Facebook App.

8. **Website form**
   Connect via a webhook URL. Form submits POST request to n8n, data maps to the same ticket context fields as Gmail.

**Phase 3 - analytics and reporting:**

9. **Weekly SLA breach report**
   Schedule a cron workflow every Monday. Reads all tickets from the past week, sends data to AI with a prompt to find patterns by category, hour, and owner. Posts a plain-language summary to Slack.

10. **Slack dashboard**
    Add a weekly digest: total tickets, breakdown by category and priority, average response time, breach rate. Gives the owner visibility into support health without opening Sheets.

---

## Part of HUSH Automation Suite

This project is #3 of an ongoing AI automation series for HUSH:
#1 — Email Support Automation (classification, auto-reply, analytics)
#2 — Customer Signal AI (call QA scoring, retention emails)
#3 — Ticket Routing with SLA Escalation (this project)
