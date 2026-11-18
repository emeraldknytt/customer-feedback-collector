# Customer Feedback Collector with Negative-Review Alerts

An n8n automation that collects customer feedback through an online form, logs every response to Google Sheets, sends each customer a follow-up email tailored to their rating, and **alerts the business owner on Telegram within seconds when a customer is unhappy**.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Project Objectives](#project-objectives)
- [Technologies and Tools](#technologies-and-tools)
- [How the Automation Works](#how-the-automation-works)
- [Workflow Architecture](#workflow-architecture)
- [Google Sheets Structure](#google-sheets-structure)
- [Installation and Setup](#installation-and-setup)
- [Customizing the Workflow](#customizing-the-workflow)
- [Testing and Validation](#testing-and-validation)
- [Error Handling Design](#error-handling-design)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Current Project Status](#current-project-status)
- [Possible Future Improvements](#possible-future-improvements)
- [Repository Structure](#repository-structure)
- [Screenshots](#screenshots)
- [Project Summary](#project-summary)

---

## Project Overview

This project is a complete customer feedback pipeline for small businesses such as shops, repair services, salons, restaurants and online sellers.

A customer opens a short feedback form and rates their experience from 1 to 5. The automation then:

1. **Cleans and enriches** the response (unique ID, timestamp, sentiment category).
2. **Saves it** to a Google Sheet that acts as the feedback database.
3. **Routes it** by sentiment: Positive, Neutral or Negative.
4. **Replies to the customer** with an email that matches their experience.
5. **Alerts the owner on Telegram** immediately if the feedback is negative, and records that the alert was sent.

A separate **error-handling workflow** emails the owner if anything in the main workflow fails, so problems are never silent.

---

## Problem Statement

Many small businesses collect feedback but rarely act on it in time:

- Feedback sits unread in a spreadsheet or inbox.
- Unhappy customers are not contacted quickly, so they leave or complain publicly.
- Customers who take time to give feedback receive no acknowledgement.
- When a manual process breaks, nobody notices.

The cost is lost customers and damaged reputation, often from problems that could have been fixed with a quick follow-up.

---

## Project Objectives

| # | Objective | Status |
|---|---|---|
| 1 | Collect feedback through a simple, validated online form | ✅ Implemented & tested |
| 2 | Assign each response a unique ID, timestamp and sentiment category | ✅ Implemented & tested |
| 3 | Store every response in Google Sheets before any other action | ✅ Implemented & tested |
| 4 | Send each customer a sentiment-appropriate email | ✅ Implemented & tested |
| 5 | Alert the owner on Telegram instantly for negative feedback | ✅ Implemented & tested |
| 6 | Record in the sheet whether the owner alert was delivered | ✅ Implemented & tested |
| 7 | Notify the owner by email when the workflow fails | ✅ Implemented & tested |

---

## Technologies and Tools

| Tool | Role in the Project | Pricing / Availability |
|---|---|---|
| **n8n** | Workflow automation platform that runs both workflows | n8n Cloud (14-day free trial, then paid) or free self-hosted Community Edition |
| **n8n Form Trigger** | Hosts the customer feedback form and starts the workflow on submission | Included in n8n |
| **Google Sheets** | Feedback database and owner's follow-up tracker | Free with a Google account |
| **Gmail** | Sends customer follow-up emails and owner error notifications | Free with a Google account |
| **Telegram Bot API** | Delivers instant negative-feedback alerts to the owner's phone | Free; bot created via @BotFather |

**Key n8n concepts used:** Form Trigger, Edit Fields (Set), Google Sheets Append Row / Update Row, Switch routing with a fallback output, cross-node expressions (`$('Node Name')`), per-node error settings (*On Error → Continue*), Error Trigger workflows, and workflow timezone settings.

---

## How the Automation Works

### Main workflow: `Customer Feedback Collector`

**1. Customer submits the form.** The form collects Customer Name, Email, Product or Service, Rating (1–5 dropdown) and optional Comments. The form itself rejects missing required fields and invalid email formats, and the dropdown makes out-of-range ratings impossible.

**2. Data is cleaned and enriched** (`Prepare Feedback Data`):

- Trims accidental spaces from text fields and lowercases the email address.
- Converts the rating from text to a number.
- Replaces an empty comment with `No comment provided`.
- Generates a unique ID in the format `FB-YYYYMMDD-HHMMSS-XXX`.
- Adds a local timestamp.
- Classifies sentiment and sets the initial status:

| Rating | Sentiment | Initial Status |
|---|---|---|
| 4–5 | Positive | No action needed |
| 3 | Neutral | No action needed |
| 1–2 | Negative | Follow-up needed |

**3. The response is logged first** (`Log Feedback to Sheet`). Saving happens **before** any routing, so the feedback is recorded even if a later step fails.

**4. The response is routed by sentiment** (`Route by Sentiment`):

| Branch | Actions |
|---|---|
| **Positive** | Thank-you email to the customer |
| **Neutral** | Email inviting the customer to reply with suggestions |
| **Negative** | Apology email → Telegram alert to the owner → sheet updated to `Alert Sent = Yes` |
| **Fallback** | Safety net for any unexpected value; such items stop here and are visible in the execution log |

### Error workflow: `Feedback Collector - Error Handler`

If any run of the main workflow fails in production, n8n automatically starts this workflow. It emails the owner the workflow name, the failed node, the error message, the time and a link to the failed execution.

---

## Workflow Architecture

### Main workflow

```
┌──────────────────────────┐
│  Feedback Form           │  n8n Form Trigger
│  (customer submits)      │
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Prepare Feedback Data   │  Edit Fields: clean, ID, timestamp,
│                          │  sentiment, initial status
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Log Feedback to Sheet   │  Google Sheets: Append Row
└────────────┬─────────────┘
             ▼
┌──────────────────────────┐
│  Route by Sentiment      │  Switch (3 rules + fallback)
└──┬─────────┬─────────┬───┘
   │         │         │
Positive  Neutral   Negative
   │         │         │
   ▼         ▼         ▼
┌────────┐┌────────┐┌──────────────────┐
│ Email: ││ Email: ││ Email: Apology   │  (On Error: Continue)
│ Thank  ││Request ││                  │
│ You    ││Suggest-│└────────┬─────────┘
│        ││ ions   │         ▼
└────────┘└────────┘┌──────────────────┐
                    │ Alert Owner on   │  Telegram: Send Message
                    │ Telegram         │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Mark Alert as    │  Google Sheets: Update Row
                    │ Sent             │  (match on Feedback ID)
                    └──────────────────┘
```

### Error workflow

```
┌──────────────────────┐      ┌──────────────────────────────┐
│  On Workflow Error   │ ───▶ │  Email Owner: Workflow Failed │
│  (Error Trigger)     │      │  (Gmail)                      │
└──────────────────────┘      └──────────────────────────────┘
```

### Node reference

| Node | Type | Purpose |
|---|---|---|
| Feedback Form | n8n Form Trigger | Hosts the form; starts the workflow on submission |
| Prepare Feedback Data | Edit Fields (Set) | Cleans input; adds ID, timestamp, sentiment, status |
| Log Feedback to Sheet | Google Sheets (Append Row) | Stores every response |
| Route by Sentiment | Switch | Sends each response down the correct branch |
| Email: Thank You | Gmail | Positive follow-up |
| Email: Request Suggestions | Gmail | Neutral follow-up |
| Email: Apology | Gmail | Negative follow-up with a 24-hour contact promise |
| Alert Owner on Telegram | Telegram | Instant negative-feedback alert |
| Mark Alert as Sent | Google Sheets (Update Row) | Sets `Alert Sent = Yes` after a successful alert |
| On Workflow Error | Error Trigger | Receives failure details from the main workflow |
| Email Owner: Workflow Failed | Gmail | Notifies the owner of the failure |

### Design decisions

| Decision | Reason |
|---|---|
| n8n Form Trigger instead of Google Forms | Runs instantly on submission with no polling, and validates input at the form level |
| Save before routing | Feedback is never lost, even if an email or alert fails |
| Data referenced via `$('Prepare Feedback Data')` | Each node changes what `$json` contains; referencing the source node keeps values reliable throughout every branch |
| Apology email before the Telegram alert | The customer always receives a reply, even if the owner alert fails |
| Sheet updated only after the alert succeeds | `Alert Sent = No` on a negative row is visible evidence that the alert failed |
| Error notifications by email, not Telegram | The error channel must not depend on the service most likely to have failed |

---

## Google Sheets Structure

Create a spreadsheet (e.g. `Customer Feedback Log`) with a tab named **`Feedback`** and these headers in **row 1**, exactly as written:

| Column | Header | Example | Filled by |
|---|---|---|---|
| A | `Feedback ID` | `FB-20261117-225611-159` | Workflow |
| B | `Timestamp` | `2026-11-17 22:56:11` | Workflow |
| C | `Customer Name` | `Tunde Bakare` | Customer |
| D | `Email` | `tunde@example.com` | Customer |
| E | `Product/Service` | `iPhone 13 repair` | Customer |
| F | `Rating` | `2` | Customer |
| G | `Comments` | `No comment provided` | Customer / Workflow |
| H | `Sentiment` | `Negative` | Workflow |
| I | `Alert Sent` | `Yes` | Workflow |
| J | `Status` | `Follow-up needed` | Workflow, then the owner |

> **Note:** The workflow maps data to columns by header name. A typo or extra space in a header will break the mapping.

The `Status` column doubles as a follow-up tracker: the owner can change `Follow-up needed` to `Resolved` once the customer has been contacted.

---

## Installation and Setup

### Prerequisites

- An n8n account: [n8n Cloud](https://n8n.io) (14-day free trial) or a self-hosted n8n instance
- A Google account (for Google Sheets and Gmail)
- A Telegram account

> On n8n Cloud, Google accounts connect with a one-click sign-in. On a self-hosted instance, you will need to create your own Google Cloud OAuth credentials. See the [n8n Google credentials documentation](https://docs.n8n.io/integrations/builtin/credentials/google/).

### Step 1: Get the workflow files

Clone or download this repository. The workflows are in the `workflows/` folder:

- `customer-feedback-collector.json` (main workflow)
- `feedback-collector-error-handler.json` (error workflow)

### Step 2: Create the Google Sheet

Create the spreadsheet and `Feedback` tab described in [Google Sheets Structure](#google-sheets-structure).

### Step 3: Create a Telegram bot and get your chat ID

1. In Telegram, open **@BotFather**, send `/newbot` and follow the prompts.
2. Copy the **bot token** BotFather gives you. Keep it private.
3. Open a chat with your new bot and send it any message (a bot cannot message you first).
4. Visit `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in a browser and find the number after `"chat":{"id":`. This is your **chat ID**.

### Step 4: Import both workflows into n8n

For each JSON file: create a new workflow in n8n, open the workflow menu, and choose **Import from File**. Import the error handler first.

### Step 5: Connect your credentials

Create these credentials in n8n and select them in the matching nodes:

| Credential | Used by | What you need |
|---|---|---|
| Google Sheets OAuth2 API | `Log Feedback to Sheet`, `Mark Alert as Sent` | Sign in with Google and grant all requested permissions |
| Gmail OAuth2 API | All four Gmail nodes | Sign in with Google and grant all requested permissions |
| Telegram API | `Alert Owner on Telegram` | Your bot token |

Imported workflows do **not** contain credentials. You must select your own in every node that shows a credential warning.

### Step 6: Configure your values

Work through the [Customizing the Workflow](#customizing-the-workflow) table. At minimum, set your **Google Sheet**, **Telegram chat ID**, **owner email** and **business name**.

### Step 7: Link the error workflow

Open the main workflow → **Settings** → set **Error Workflow** to `Feedback Collector - Error Handler` → save. This link must be set again after importing, because it references a workflow ID that is unique to each n8n instance.

### Step 8: Set the timezone

In each workflow's **Settings**, set **Timezone** to your local timezone (this project uses `Africa/Lagos`). IDs and timestamps are generated in this timezone.

### Step 9: Go live

1. Activate or publish the main workflow (the control depends on your n8n version).
2. Open the `Feedback Form` node and copy the **Production URL**.
3. Share this URL with customers: in emails, on receipts, as a QR code, or on your website.

> The **Test URL** only works while the editor is waiting for a test submission. Customers must always receive the **Production URL**.

---

## Customizing the Workflow

| What to change | Where | Details |
|---|---|---|
| **Google Sheet** | `Log Feedback to Sheet` and `Mark Alert as Sent` → Document / Sheet | Select your own spreadsheet and the `Feedback` tab |
| **Telegram chat ID** | `Alert Owner on Telegram` → Chat ID | Replace `YOUR_TELEGRAM_CHAT_ID` with your chat ID |
| **Owner email address** | `Email Owner: Workflow Failed` (error workflow) → To | Replace `owner@example.com` with the address that should receive failure alerts |
| **Business name** | The signature in all three customer emails and the optional **Sender Name** option on each Gmail node | Replace `Your Business Name` |
| **Email wording** | `Email: Thank You`, `Email: Request Suggestions`, `Email: Apology` → Subject / Message | Edit the HTML. Keep the `{{ }}` expressions to personalize each email |
| **Follow-up promise** | `Email: Apology` and `Alert Owner on Telegram` | Both mention **24 hours**; change both together so they match |
| **Form title, description and fields** | `Feedback Form` | If you rename a field label, update the matching expression in `Prepare Feedback Data` (e.g. `$json['Customer Name']`) |
| **Sentiment thresholds** | `Prepare Feedback Data` → `sentiment` and `status` fields | Default: 4–5 Positive, 3 Neutral, 1–2 Negative |
| **Feedback ID format** | `Prepare Feedback Data` → `feedbackId` | Default prefix `FB-` followed by date, time and a random 3-digit number |
| **Timezone** | Each workflow's Settings | Default `Africa/Lagos` |

**Example: making a rating of 3 count as negative.** In `Prepare Feedback Data`, change the `status` condition from `<= 2` to `<= 3`, and change the `sentiment` expression so that ratings of 3 or below return `Negative`.

---

## Testing and Validation

All tests were run with realistic sample data. Tests 1–6 were run in n8n's test mode. Tests 7–9 were run against the live **Production URL**, because n8n error workflows only run for production executions.

| # | Scenario | Input | Expected Result | Result |
|---|---|---|---|---|
| 1 | Positive feedback | Ratings 5 and 4 | Row logged; thank-you email sent; no alert | ✅ Passed |
| 2 | Neutral feedback | Rating 3 | Row logged; suggestions email sent; no alert | ✅ Passed |
| 3 | Negative feedback | Ratings 2 and 1 | Row logged; apology email; Telegram alert; `Alert Sent = Yes` with the rest of the row intact | ✅ Passed |
| 4 | Optional comment left blank | Rating 2, no comment | Comment stored as `No comment provided` | ✅ Passed |
| 5 | Missing required field | No customer name | Form blocks submission | ✅ Passed |
| 6 | Invalid email format | `tunde@` | Form blocks submission | ✅ Passed |
| 7 | Production end-to-end | Rating 5 via Production URL | Successful live execution; email received | ✅ Passed |
| 8 | Telegram failure | Invalid chat ID; rating 1 | Row logged with `Alert Sent = No`; apology still sent; error workflow emails owner | ✅ Passed |
| 9 | Customer email failure | Invalid recipient; rating 2 | Workflow continues; Telegram alert sent; `Alert Sent = Yes`; no error email | ✅ Passed |

Boundary values were covered: rating **2** routed to Negative, rating **3** to Neutral and rating **4** to Positive.

---

## Error Handling Design

The workflow is protected in five layers:

| Layer | Mechanism | What it protects against |
|---|---|---|
| 1. Input validation | Required fields, Email field type and rating dropdown on the form | Missing data, invalid emails, out-of-range ratings |
| 2. Save first | Google Sheets logging runs before any routing | Losing feedback when a later step fails |
| 3. Resilient customer emails | *On Error → Continue* on all customer Gmail nodes | One bad email address blocking the owner's alert |
| 4. Delivery audit trail | `Alert Sent` changes to `Yes` only after the Telegram alert succeeds | Failed alerts going unnoticed |
| 5. Error workflow | Error Trigger → Gmail notification to the owner | Any unhandled failure in a production run |

The Switch node also has a **Fallback** output, so an item with an unexpected sentiment value appears in the execution log instead of disappearing without a trace.

---


## Project Summary

The **Customer Feedback Collector** turns customer feedback into immediate, trackable action. Every response is validated, stored and acknowledged. Unhappy customers are flagged to the owner within seconds, and failures are reported instead of passing unnoticed.

The project demonstrates practical automation skills: form-based triggers, data cleaning with expressions, spreadsheet storage and row updates, conditional routing, multi-channel notifications, defensive error handling, and systematic testing with both success and failure cases.

---

*Built with [n8n](https://n8n.io).*
