# 📧 Product Team Email Summarizer

An automated email triage tool for the Wowcher product team. Fetches emails from a shared Outlook mailbox, classifies them using OpenAI GPT-4.1-mini, and appends results to a shared SharePoint Excel file. The team is notified by email after every update.

---

## How It Works

1. **Fetches** only new emails from `inbox` since the last run (incremental scan)
2. **Filters** out automated/irrelevant emails (e.g. no-reply senders, release requests)
3. **Classifies** each email using OpenAI - assigns category, urgency, summary and owner
4. **Appends** results to a shared Excel file on SharePoint (sorted by urgency)
5. **Notifies** the team by email with a direct link to the updated file

---

## Features

- **Incremental scanning** — only processes new emails since the last run via `last_run.txt`
- **AI classification** — OpenAI GPT-4.1-mini assigns category, urgency, summary, owner and confidence
- **VIP stakeholder detection** — emails from named stakeholders are always flagged High urgency with the stakeholder named explicitly in the reasoning column
- **Smart filtering** — ignores automated senders and emails containing "release request" in subject or body
- **Adaptive rate limiting** — automatically slows down and retries if API limits are hit (5 retries with exponential backoff)
- **SharePoint integration** — downloads, updates and re-uploads the shared Excel file each run
- **Email notifications** — notifies the team with a clickable link to the file after every update
- **Performance metrics** — logs success rate, time taken and rate limit hits after each run

---

## Excel Output

All classified emails are appended to `Email summary.xlsx` on SharePoint, sorted by urgency:

| Column | Description |
|--------|-------------|
| Date / Time | When the email was received |
| Sender | Display name of sender |
| Subject | Email subject |
| Category | AI-assigned issue type |
| New? | Whether it's a new category not seen before |
| Urgency | High 🔴 / Medium 🟡 / Low 🟢 |
| Summary | Actionable 2-sentence summary starting with a verb |
| Assigned | product team members
| Confidence | High / Medium / Low |
| Reasoning | Why this urgency was chosen (names stakeholder if applicable) |
| Link | Direct link to the email in Outlook |

### Email Categories

| Category | Description |
|----------|-------------|
| Calendar Issues | Travel deals, booking calendars, channel manager integrations |
| Redemption Issues | Voucher codes not working, redemption failures |
| SQL Updates | Database changes, bulk updates, salesperson changes |
| Pricing Issues | Wrong prices, discount calculation errors |
| Data Sync Issues | Channel manager sync failures, inventory sync problems |
| *(new)* | AI will create new categories for issue types not listed above |

### Urgency Rules

| Urgency | Triggered By |
|---------|-------------|
| 🔴 High | VIP stakeholder email, customer-facing issue, urgent/ASAP/critical keywords, email >2 days old, SQL updates |
| 🟡 Medium | Internal issues, non-urgent requests |
| 🟢 Low | Informational emails, already resolved |

### VIP Stakeholders (always High urgency)
Stakeholder names

---

## Prerequisites

### Python packages
```bash
pip install msal requests openai openpyxl
```

### Azure AD App Permissions
The following **Application permissions** must be granted on the Azure AD app (`bec11ba8-2416-49c7-8135-698cd1c8b11f`) via IT:

| Permission | Purpose |
|------------|---------|
| `Mail.Read` | Read emails from the shared mailbox |
| `Files.ReadWrite` | Download and upload the SharePoint Excel file |
| `Mail.Send` | Send email notifications to the team |

---

## Configuration

All settings are at the top of `email_summarizer_merged.py`:

```python
# Azure AD (set via environment variables in production)
CLIENT_ID        = 'your-azure-app-client-id'
CLIENT_SECRET    = 'your-azure-app-client-secret'
TENANT_ID        = 'your-azure-tenant-id'
MAILBOX_EMAIL    = 'your email inbox'

# OpenAI
OPENAI_API_KEY   = 'your-openai-api-key'   # ← must be set

# SharePoint
SHAREPOINT_FILE_ID  = 'your-sharepoint-file-id'
SHAREPOINT_USER     = 'your_name_wowcher_co_uk'

# Notification recipients
NOTIFY_RECIPIENTS = [
    'your emails'
]
```

### Environment Variables (recommended for production)

Rather than hardcoding credentials, set these environment variables:

```bash
export AZURE_CLIENT_ID="..."
export AZURE_CLIENT_SECRET="..."
export AZURE_TENANT_ID="..."
export OPENAI_API_KEY="..."
export MAILBOX_EMAIL="your email inbox"
export OUTPUT_DIR="/path/to/output"
export DAYS_TO_SCAN="7"   # Only used on first run
```

---

## Adding Filters

### Ignore a sender
In `email_summarizer_merged.py`, add to `IGNORED_SENDERS`:
```python
IGNORED_SENDERS = [
    'no-reply@travelgate.com',
    'notifications@jira.com',   # ← add here
]
```

### Ignore emails containing a phrase
Add to `IGNORED_SUBJECTS` (checks subject AND full body):
```python
IGNORED_SUBJECTS = [
    'release request',
    'out of office',   # ← add here
]
```

---

## Running Locally

```bash
# Install dependencies
pip install msal requests openai openpyxl

# Set your OpenAI key
export OPENAI_API_KEY="sk-..."

# Run
python email_summarizer_merged.py
```

**First run:** scans the last 7 days and creates `last_run.txt`

**Subsequent runs:** only processes emails received since the last run

---

## Deployment (Airflow)

The script is designed to run on a schedule via Apache Airflow. Hand `email_summarizer_dag.py` to engineering with your chosen schedule.

Example schedules:
```python
# 4x daily Mon-Fri
schedule_interval='0 7,10,13,16 * * MON-FRI'

# Every 2 hours Mon-Fri
schedule_interval='0 */2 8-18 * * MON-FRI'

# Every hour Mon-Fri
schedule_interval='0 9-17 * * MON-FRI'
```

---

## Files

| File | Description |
|------|-------------|
| `email_summarizer_merged.py` | Main production script |
| `email_summarizer_dag.py` | Airflow DAG for scheduling |
| `last_run.txt` | Auto-generated — stores timestamp of last successful run. Do not delete. |
| `README.md` | This file |

---

## Troubleshooting

**403 Authentication error**
Azure AD app is missing a required permission. Check IT have granted `Mail.Read`, `Files.ReadWrite` and `Mail.Send` as Application permissions and clicked "Grant admin consent".

**Rate limit warnings in logs**
The adaptive rate limiter will handle this automatically with retries. If it persists, increase `INITIAL_DELAY` at the top of the script from `0.5` to `1.5`.

**Emails missing from spreadsheet**
Check `last_run.txt` — if it has an incorrect timestamp, delete the file and re-run to trigger a full 7-day scan.

**SharePoint upload failing**
Confirm `SHAREPOINT_FILE_ID` matches the file ID in the SharePoint URL and that `Files.ReadWrite` permission has been granted.

---

## Built With

- [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/overview) — email and SharePoint access
- [OpenAI GPT-4.1-mini](https://platform.openai.com/docs/) — email classification
- [MSAL](https://github.com/AzureAD/microsoft-authentication-library-for-python) — Azure AD authentication
- [openpyxl](https://openpyxl.readthedocs.io/) — Excel file generation
