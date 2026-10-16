# AI Lead Qualification System (Dubai Real Estate)

An n8n workflow that turns free-text property enquiries into structured,
scored leads and notifies the sales team instantly. It handles English,
Arabic and mixed-language messages and normalises budgets to AED.

**Demo video:** [add link]
**Dashboard (view-only):** [add Looker Studio link]

## The problem

Real-estate agencies get enquiries through forms and messages in
inconsistent formats. Sales teams read each one manually, copy details
into a sheet, and decide who to call first. This is slow, and hot leads
go cold while they wait.

## What it does

1. A lead submits an enquiry through a web form (email + free-text message).
2. Gemini extracts: name, location, budget (AED), bedrooms, property
   type, intent, priority and language.
3. A rule-based scorer classifies the lead as **Hot / Warm / Cold**.
4. The lead is logged to Google Sheets.
5. The lead gets an instant, personalised auto-reply (bilingual if the
   enquiry was in Arabic).
6. Hot and Warm leads trigger a Slack alert and an email to sales.
7. Invalid or unparseable outputs go to a "Needs Review" sheet with a
   Slack warning. A separate error workflow alerts on workflow failures.

## Architecture

[Add architecture diagram image here: screenshots/workflow-canvas.png]

```
Form -> Gemini extraction -> Parse + validate + score -> Valid?
   no  -> Needs Review sheet + Slack warning
   yes -> Google Sheets log
       -> Auto-reply email to lead
       -> Hot/Warm? -> Slack alert + Sales email
```

## Tech stack

n8n Cloud, Google Gemini API, Google Sheets, Gmail, Slack (incoming
webhook), Looker Studio

## Key features

- **Structured extraction:** LLM returns strict JSON, validated in code.
- **Deterministic scoring:** transparent points system instead of an
  opaque LLM verdict (see below), with an LLM-written reason for context.
- **Dubai-specific handling:** Arabic and English input, budgets such as
  "2M", "800k" or "$500k" converted to AED, area names standardised
  (JVC, Dubai Marina, Business Bay, etc.).
- **Error handling:** retry on LLM failure, JSON validation, a review
  queue for bad outputs, and a global error workflow.
- **Dashboard:** leads by score, area, budget and day.

## Scoring rules

| Factor | Points |
|---|---|
| Budget >= AED 2,000,000 | 3 |
| Budget >= AED 800,000 | 2 |
| Budget stated (below 800,000) | 1 |
| Priority: high / medium / low | 3 / 2 / 0 |
| Location stated | 1 |
| Bedrooms stated | 1 |

**Hot** = 6+ points, **Warm** = 4-5, **Cold** = under 4.
For rentals, yearly rent is multiplied by 10 as a rough purchase-equivalent.
These thresholds are illustrative and should be tuned per agency.

## Extracted JSON schema

```json
{
  "name": "string | null",
  "location": "string | null",
  "budget_aed": "number | null",
  "budget_text": "string | null",
  "bedrooms": "integer | null",
  "property_type": "apartment | villa | townhouse | penthouse | studio | other | null",
  "intent": "buy | rent | invest | unknown",
  "priority": "high | medium | low",
  "language": "en | ar | mixed | other",
  "summary": "string",
  "score_reason": "string"
}
```

The full prompt is in [`prompts/extraction-prompt.md`](prompts/extraction-prompt.md).

## Setup

**Prerequisites:** n8n (Cloud or self-hosted), a Google account,
a Gemini API key from Google AI Studio, a Slack workspace.

1. **Google Sheet:** create a sheet with two tabs.
   - `Leads`: columns `timestamp, name, email, phone, message, location,
     budget_aed, budget_text, bedrooms, property_type, intent, priority,
     score, points, reason, summary, language, status`
   - `Needs Review`: columns `timestamp, email, message, error, raw_output`
2. **Slack:** create an app with Incoming Webhooks enabled and add a
   webhook for your leads channel.
3. **Import:** in n8n, go to Workflows -> Import from file and import
   `workflow.json`, then `error-handler.json`.
4. **Credentials:** create and select your own Gemini, Google Sheets
   and Gmail credentials in the relevant nodes.
5. **Placeholders:** replace these values in the imported workflow.
   - Both Slack HTTP Request nodes: paste your webhook URL
     (placeholder: `https://hooks.slack.com/services/YOUR/WEBHOOK/URL`)
   - Sheets nodes: select your own spreadsheet
   - Notify sales node: set the recipient email
6. **Error workflow:** in the main workflow's Settings, set
   Error Workflow to the imported error handler.
7. **Activate** the workflow and open the production form URL.

## Testing

Sample enquiries (English, Arabic, rental, USD investor, vague, empty)
are in [`sample-leads.md`](sample-leads.md), with the output I observed
for each.

## Results

[Fill in after your own timing test, for example:]
Manual processing of 10 test leads took [X] minutes; the workflow
processed the same leads in [Y] seconds. Measured on synthetic data.

## Limitations

- Scoring thresholds are heuristic, not trained on real conversion data.
- Only USD and AED are converted; other currencies are flagged as null.
- Tested with synthetic data only; the Gemini free tier is not suitable
  for real customer data.
- No duplicate detection yet.

## Future improvements

- Duplicate-lead detection by email/phone
- Property matching against a listings sheet
- Escalation reminder if a Hot lead isn't contacted within X hours
- WhatsApp intake

## Author

Nupur Khapali
