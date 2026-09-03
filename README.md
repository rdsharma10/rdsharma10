# Gmail to Gemini AI to Salesforce     

A Node.js service that watches internal Gmail mailboxes in real time, uses Google Gemini AI to analyze inbound client conversations, and pushes qualified leads into Salesforce via a custom Apex REST endpoint.

Built for HIC Global Solutions / Task Optimizer App.

| | |
|---|---|
| Runtime | Node.js 18+ |
| AI Engine | Google Gemini (model fallback chain) |
| Trigger | Gmail API + Google Cloud Pub/Sub (real-time push) |
| Destination | Salesforce (custom Apex REST endpoint) |
| Status | Actively maintained |

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Modules](#core-modules)
4. [Key Features](#key-features)
5. [Setup](#setup)
6. [Google Cloud Pub/Sub Setup](#google-cloud-pubsub-setup)
7. [Domain-Wide Delegation](#google-workspace-admin-console--domain-wide-delegation)
8. [Install and Run](#install-and-run)
9. [Configuration](#configuration)
10. [Known Operational Notes](#known-operational-notes)
11. [Status](#status)

---

## Overview

| Step | Action |
|:---:|---|
| 1 | A Gmail Pub/Sub watch is set up on each internal mailbox. |
| 2 | A new email arrives, and a Pub/Sub push notification triggers the service. |
| 3 | The service fetches the full email thread with the external sender, builds a conversation snapshot, and sends it to Gemini AI for analysis: promotional detection, lead scoring, buyer intent, product/pricing extraction, etc. |
| 4 | The resulting structured payload is sent to Salesforce through a custom Apex REST endpoint (`/ai/lead`). |
| 5 | Promotional senders are auto-blocked; internal/HIC domains are ignored; group mailbox leads get an owner assigned automatically. |

---

## Architecture

```
                    Gmail (per-user watch)
                            |
                            v
             Google Cloud Pub/Sub (push subscription)
                            |
                            v
        index.js  -->  fetch history / conversation thread
                            |
                            v
   sendToSalesforce.js  -->  gemini.js  (AI analysis)
                            |
                            v
            Salesforce Apex REST  (/ai/lead)
```

---

## Core Modules

| File | Responsibility |
|------|-----------------|
| `index.js` | Pub/Sub listener, Gmail history diffing, conversation fetching, group/owner resolution, processing queue |
| `sendToSalesforce.js` | Runs Gemini analysis, applies domain/promotional filtering, pushes payload to Salesforce |
| `gemini.js` | Calls Gemini API with model fallback chain, returns structured lead payload |
| `config.js` | Runtime config store — HIC domains, public email domains, hard-blocked domains |
| `gmailWatch.js` | Registers/renews Gmail watch subscriptions for all internal users |
| `SalesforceUsers.js` | Fetches the list of internal user emails from Salesforce |
| `Hardcodedomains.js` | Fetches blocked/HIC domain lists from Salesforce |

---

## Key Features

- Real-time processing via Gmail Pub/Sub push notifications (no polling)
- Per-user watch renewal every 6 hours via `node-cron`
- History-based diffing (`historyId`) — only new messages are processed after startup seeding
- Deduplication — `Message-ID` header prevents double-processing when the same email lands in multiple mailboxes (e.g. group aliases)
- Per-user locking (`withUserLock`) to avoid race conditions when multiple Pub/Sub events arrive for the same mailbox concurrently
- AI-powered lead analysis via Gemini, with a model fallback chain (`gemini-2.5-flash-lite -> gemini-2.5-flash -> gemini-2.5-pro`) for reliability
- Group mailbox owner resolution:
  - New client thread on a group address (e.g. `all@`, `support@`) is assigned to a domain-based default owner
  - If any internal user replies to the client, ownership automatically transfers to that user on the next inbound message
- Domain filtering:
  - Internal/company domains are ignored entirely (no lead created)
  - Public domains (Gmail, Yahoo, Outlook, etc.) are never auto-blocked, even if flagged promotional
  - Non-public promotional domains are auto-added to Salesforce's `Hard_Blocked_Domains__c`
- IST date formatting — Gemini returns `First Message Date` / `Latest Message Date` in strict `MMM DD, YYYY hh:mm AM/PM IST` format for consistent Salesforce storage

---

## Setup

### Prerequisites

- Node.js 18+
- A Google Cloud project with:
  - Gmail API enabled
  - Pub/Sub topic + push subscription configured
  - A service account with domain-wide delegation and `gmail.readonly` scope
- Salesforce org with:
  - A custom Apex REST endpoint (`/ai/lead`)
  - Custom object `Hard_Blocked_Domains__c` — fields: `Name`, `Domain__c`, `Is_HIC_Domain__c`
- Gemini API key

### Environment Variables (`.env`)

```dotenv
SF_LOGIN_URL=https://login.salesforce.com
SF_USERNAME=your_sf_username
SF_PASSWORD=your_sf_password_and_token
PUBSUB_SUBSCRIPTION=your-pubsub-subscription-name
GEMINI_API_KEY=your_gemini_api_key
```

Also required: `service-account.json` (Google service account key file) in the project root.

---

## Google Cloud Pub/Sub Setup

Gmail can only notify you of new mail through a Pub/Sub topic — this must be created and wired up before `gmail.users.watch()` (used in `gmailWatch.js`) will work.

### 1. Create a Pub/Sub topic

```bash
gcloud pubsub topics create gmail-notifications
```

### 2. Grant Gmail permission to publish to the topic

Gmail's push service account must be allowed to publish:

```bash
gcloud pubsub topics add-iam-policy-binding gmail-notifications \
  --member="serviceAccount:gmail-api-push@system.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

### 3. Create a push subscription

This is the subscription name that goes into `PUBSUB_SUBSCRIPTION` in `.env`.

```bash
gcloud pubsub subscriptions create gmail-notifications-sub \
  --topic=gmail-notifications \
  --ack-deadline=60
```

Note: the project uses the pull-style client (`@google-cloud/pubsub`'s `subscription.on('message', ...)`), so a push endpoint/URL is not required — the subscription just needs to exist, and the service account running `index.js` needs `roles/pubsub.subscriber` on it.

### 4. Grant the app's service account subscriber access

```bash
gcloud pubsub subscriptions add-iam-policy-binding gmail-notifications-sub \
  --member="serviceAccount:YOUR_SERVICE_ACCOUNT_EMAIL" \
  --role="roles/pubsub.subscriber"
```

### 5. Register the watch per mailbox

`gmailWatch.js` calls `gmail.users.watch()` for every internal user, pointing at the topic:

```js
{
  topicName: 'projects/YOUR_PROJECT_ID/topics/gmail-notifications',
  labelIds: ['INBOX'],
  labelFilterAction: 'include'
}
```

Important — watch expiry: Gmail watches expire after 7 days. This is why `index.js` runs a cron job every 6 hours to renew all watches:

```js
cron.schedule('0 0 */6 * *', async () => {
  await startAllWatches();
});
```

### 6. Verify

Send a test email to a watched mailbox and check the logs — you should see:

```
New Gmail event for: <user>
```

---

## Google Workspace Admin Console — Domain-Wide Delegation

The service account impersonates each internal mailbox (`subject: userEmail` in `getGmailClient()`), which requires domain-wide delegation to be explicitly authorized by a Workspace super admin. Without this step, every Gmail API call fails with `unauthorized_client` or `invalid_grant`.

### 1. Get the service account's Client ID

Open `service-account.json` and copy the `client_id` value, or find it in:

Google Cloud Console -> IAM & Admin -> Service Accounts -> (your service account) -> Details

### 2. Authorize the Client ID in Workspace Admin Console

| Step | Action |
|:---:|---|
| 1 | Go to admin.google.com |
| 2 | Navigate to Security -> Access and data control -> API Controls |
| 3 | Click "Manage Domain Wide Delegation" |
| 4 | Click "Add new" |
| 5 | Enter Client ID (from step 1 above) and OAuth Scope: `https://www.googleapis.com/auth/gmail.readonly` |
| 6 | Click "Authorize" |

### 3. Propagation delay

Changes here are not always instant — they can take anywhere from a few minutes up to approximately 24 hours to fully propagate across Google's servers. This is a common cause of intermittent `unauthorized_client` errors right after setup or after modifying scopes (see [Known Operational Notes](#known-operational-notes)).

### 4. Scope changes

If you ever add a new Gmail scope (e.g. `gmail.send` for a future feature), you must add it to the existing Domain-Wide Delegation entry in the Admin Console — adding a scope in code alone does nothing without also authorizing it here.

### 5. Confirm it's working

```bash
node -e "
const { google } = require('googleapis');
const auth = new google.auth.JWT({
  keyFile: './service-account.json',
  scopes: ['https://www.googleapis.com/auth/gmail.readonly'],
  subject: 'someinternaluser@yourdomain.com'
});
google.gmail({ version: 'v1', auth }).users.getProfile({ userId: 'me' })
  .then(r => console.log('OK:', r.data))
  .catch(e => console.error('FAILED:', e.message));
"
```

If this prints the user's Gmail profile, delegation is correctly configured.

---

## Install and Run

```bash
npm install
node index.js
```

Recommended: run under a process manager (e.g. PM2) for auto-restart and log management.

```bash
pm2 start index.js --name gmail-lead-pipeline
pm2 logs index
```

---

## Configuration

### Group mailbox default owners

Configured per domain in `index.js`:

```js
const DEFAULT_GROUP_OWNER = 'piyush@hicglobalsolutions.com';   // Sales Admin

```

Any recipient address on one of these domains that is not a known internal user is treated as a group/shared mailbox, and gets the domain's default owner assigned until an internal user replies.

### Hard-blocked and HIC domains

Loaded at startup from Salesforce (`Hardcodedomains.js`) into `config.js`, and kept in sync as new promotional senders are detected.

---

## Known Operational Notes

| Note | Detail |
|---|---|
| Intermittent `unauthorized_client` (401) | Can occur occasionally due to Domain-Wide Delegation propagation delays or server clock drift — typically transient and self-resolves on retry |
| History seeding on startup | A `historyId` seed runs for all internal users so only genuinely new messages (post-startup) are processed — backlog is intentionally skipped |

---

## Status

Actively maintained — recent work has focused on deduplication logic, race-condition handling, group/owner resolution, and promotional domain auto-blocking.
