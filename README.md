# Lead Segmentation & AI Scoring Automation

An n8n workflow that automatically receives, validates, classifies, and AI scores inbound leads from any web form, then routes them to the right Google Sheet and notifies your sales team instantly.

![Workflow Diagram](./Lead%20Segmentation%20and%20AI%20Scoring%20Automation%20Workflow.png)

---

## The Problem This Solves

Most businesses treat every inbound lead the same way. Someone manually checks the inbox, copies data into a spreadsheet, and tries to guess which leads are worth calling first.

This workflow eliminates that entirely. A lead comes in, gets classified, scored by AI, stored in the right place, and your team gets an email alert, all before anyone opens their laptop.

---

## What It Does, Step by Step

**1. Receive Lead via Webhook**
Accepts POST requests from any web form, landing page, or frontend. Returns proper HTTP responses (200 success, 400 error) so the form knows whether the submission was processed.

**2. Validate Email Present**
Checks whether the submitted data contains an email address before any processing begins. If the email is missing, the incomplete submission is logged to a separate sheet and a 400 error is returned immediately. No junk data enters the pipeline.

**3. Classify Lead by Domain**
A JavaScript code node extracts the email domain and checks it against a list of known free and personal email providers:

```
gmail.com, yahoo.com, outlook.com, hotmail.com,
protonmail.com, aol.com, icloud.com, mail.com,
yandex.com, zoho.com
```

Free domain leads get tagged as `Personal`. Any other domain gets tagged as `Business`.

**4. Score Lead via Groq AI**
Lead details are passed to `llama-3.3-70b-versatile` via Groq's API with a structured prompt requesting a buying intent score from 1 to 10. It returns a single number, no extra explanation.

**5. Merge Score and Lead Data**
A code node merges the AI score with the original lead data (name, email, phone, lead type) into a single clean object for the nodes downstream.

**6. Route by Lead Type**
An IF node checks the lead type and routes accordingly. Personal leads go to the Personal Leads Google Sheet, business leads go to the Business Leads Google Sheet.

**7. Notify Sales Team**
A Gmail node sends an instant email alert to the sales team with name, email, phone, lead type, and AI score included.

**8. Return Success Response**
The final webhook response confirms the lead was processed, scored, and saved.

---

## Workflow Architecture

```
Webhook (POST)
    │
    ▼
Validate Email Present
    ├── Missing → Save Incomplete Lead → Return 400
    └── Present → Classify Lead by Domain
                         │
                         ▼
                  Score Lead via Groq AI
                  ├── Success → Merge Score + Lead Data
                  └── Error   → Fallback: score = 5, scored = false
                                       │
                                       ▼
                               Check if Lead is Personal
                               ├── true  → Save to Personal Leads Sheet
                               └── false → Save to Business Leads Sheet
                                                    │
                                                    ▼
                                           Notify Sales Team (Gmail)
                                                    │
                                                    ▼
                                           Return 200, Lead Processed
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n | Workflow automation engine |
| Groq API | AI inference (llama-3.3-70b-versatile) |
| Google Sheets | Lead storage (Personal and Business tabs) |
| Gmail | Sales team notification |
| JavaScript (Code node) | Domain classification and data merging |

---

## Error Handling

This workflow is built for reliability, not just demos.

| Scenario | Behavior |
|---|---|
| Email missing in submission | Logged to Incomplete Leads sheet, 400 returned |
| Invalid email format | Code node throws an error, execution stops cleanly |
| Groq API failure | Fallback branch runs, lead saved with score = 5 and `scored: false` |
| Duplicate lead (same email) | Google Sheets node uses `appendOrUpdate`, existing row is updated instead of duplicated |

---

## Google Sheets Structure

Both the Personal and Business sheets use the same column schema.

| Column | Description |
|---|---|
| name | Lead's full name |
| email | Email address, used as the unique key |
| phone | Phone number |
| Score | AI buying intent score (1 to 10) |
| LeadType | `Personal` or `Business` |
| Scored | `Yes` if AI scored successfully, `No` if the fallback was used |

---

## Setup Instructions

### 1. Import the Workflow
Download `Lead Segmentation & AI Scoring.json` and import it into your n8n instance through **Workflows → Import from file**.

### 2. Configure Credentials
You'll need to connect the following in n8n:

- **Google Sheets OAuth2**, connect your Google account
- **Gmail OAuth2**, connect the account that will send alerts
- **Groq API**, get a free API key from [console.groq.com](https://console.groq.com)

### 3. Set Up Google Sheets
Create two Google Sheets with these column headers in Row 1:

```
name | email | phone | Score | LeadType | Scored
```

Update the Google Sheets node in the workflow to point to your own sheet IDs.

### 4. Update Gmail Recipient
In the `Notify Sales Team` node, change the `sendTo` address to your sales team's email.

### 5. Activate the Workflow
Click **Publish** in n8n. Your webhook URL will be live and ready to receive POST requests.

---

## Testing

Send a test POST request using curl or Postman.

```bash
# Valid lead, Business
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Smith",
    "email": "john@acmecorp.com",
    "phone": "+1-555-0100"
  }'

# Valid lead, Personal
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jane Doe",
    "email": "jane@gmail.com",
    "phone": "+1-555-0101"
  }'

# Invalid, missing email
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "name": "No Email",
    "phone": "+1-555-0102"
  }'
```

Expected results: the first two requests land in the correct sheet with an AI score and the sales team gets an email. The third request gets logged to the incomplete leads sheet and returns a 400.

---

## Workflow File

[Lead Segmentation & AI Scoring.json](./Lead%20Segmentation%20%26%20AI%20Scoring.json)
---

## About

Built by **Muhammad Sami Ullah**, AI and workflow automation specialist.

I build n8n automations for small businesses and B2B operations teams that eliminate manual data work, reduce response time, and keep sales pipelines clean without adding headcount.

Open to freelance projects and consulting engagements.

Connect on [LinkedIn](https://www.linkedin.com/in/ihussainisami) or reach out directly.
