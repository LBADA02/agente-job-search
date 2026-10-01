# Recruitment Agent — README

## 1. Overview

This project contains two n8n workflows.

### Workflow 1 — Send Applications
- Reads recruiter contacts from Google Sheets.
- Downloads a CV from Google Drive.
- Sends application emails through Gmail.
- Updates the corresponding spreadsheet row (status, last sent date, send count).
- Waits between emails.
- Can send a summary through Telegram.

### Workflow 2 — Clean Failed-Delivery Emails
- Searches Gmail for messages from `mailer-daemon` or `postmaster`.
- Extracts email-shaped strings from each message's snippet.
- Compares the extracted addresses with the exact Google Sheets column `Email (exact as supplied)`.
- Deletes matching contact rows from the spreadsheet.
- Deletes the Gmail bounce-notification messages returned by the search.
- Can be run manually for a full cleanup.

**Important limitations of the current workflow:** the Gmail query is `from:(mailer-daemon) OR from:(postmaster)` and has no date restriction, but the Gmail node's result limit still applies. The extraction code reads message snippets only and may extract an address that is mentioned but is not the failed recipient. It does not reliably distinguish permanent failures (such as an address that does not exist) from temporary delivery failures. Inspect matches before deleting rows. Gmail's Delete operation normally moves messages to Trash rather than permanently erasing them.

## 2. Prerequisites

Install Docker Desktop. Docker Compose is included with current Docker Desktop versions. You also need:
- A Google account with access to the spreadsheet and CV
- A Gmail account used to send applications and receive bounce notifications
- A Telegram bot, if Telegram notifications are enabled

Open a terminal and verify:

```bash
docker --version
docker compose version
```

## 3. Start n8n with Docker Compose

Suggested folder structure:

```text
recruitment-agent/
├── docker-compose.yml
├── README.md
├── workflows/
│   ├── send-applications.json
│   └── clean-failed-delivery-emails.json
└── n8n_data/   # created automatically
```

Create `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - TZ=Africa/Casablanca
      - GENERIC_TIMEZONE=Africa/Casablanca
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Start n8n from the directory containing the Compose file:

```bash
docker compose up -d
docker compose ps
```

Open http://localhost:5678 and create your n8n owner account.

**Version note:** `latest` may install a newer n8n version over time. For reproducible installations, pin the same tested n8n image version as the original installation.

## 4. Configure Google OAuth credentials

Google Sheets, Google Drive, and Gmail use OAuth credentials. Each person should authorize their own Google account. Do not share personal OAuth tokens.

### 4.1 Create a Google Cloud project

1. Open https://console.cloud.google.com/
2. Create or select a project, for example `Recruitment Agent`.
3. Open **APIs & Services → Library**.

### 4.2 Enable APIs

Enable:
- Google Sheets API
- Google Drive API
- Gmail API

### 4.3 Configure OAuth consent

1. Open Google Auth Platform / OAuth consent configuration.
2. Enter the application name and contact information.
3. Configure the audience for the intended users.
4. If the app is in Testing mode, add the Google account that will authorize it as a test user.
5. Complete any Google verification or organization requirements that apply.

### 4.4 Create an OAuth client

1. Open **APIs & Services → Credentials**.
2. Select **Create Credentials → OAuth client ID**.
3. Choose **Web application**.
4. Add the authorized redirect URI shown by n8n. For a local instance at `http://localhost:5678`, this is commonly:

   `http://localhost:5678/rest/oauth2-credential/callback`

5. Create the client and copy the Client ID and Client Secret securely.

**Never put the Client Secret in this README, workflow JSON, or a public repository.** The redirect URI must match the URL shown in the n8n credential form exactly.

### 4.5 Google Sheets credential

1. In n8n, open **Credentials → Create Credential**.
2. Select the Google Sheets OAuth2 credential type.
3. Enter the OAuth Client ID and Client Secret.
4. Save and connect the Google account.
5. Authorize the account that can access the recruitment spreadsheet.
6. Name the credential `Google Sheets account`.

### 4.6 Google Drive credential

1. Create a Google Drive OAuth2 credential in n8n.
2. Use the configured OAuth client if it has the required APIs and scopes.
3. Connect the Google account containing the CV.
4. Name the credential `Google Drive account`.

### 4.7 Gmail credential

1. Create a Gmail OAuth2 credential in n8n.
2. Enter the configured OAuth Client ID and Client Secret if requested.
3. Authorize the Gmail account used to send applications and receive bounce notifications.
4. Name the credential `Gmail account`.

Troubleshooting:
- `invalid_client`: check the Client ID and Client Secret.
- Redirect URI error: match the Google Cloud URI to n8n's callback URL exactly.
- Missing files or sheets: confirm that the authorized account has access.
- Reauthorization errors: reconnect the credential and check OAuth consent/testing settings.

Official references:
- Google OAuth: https://developers.google.com/identity/protocols/oauth2
- Google Sheets API: https://developers.google.com/workspace/sheets/api
- Google Drive API: https://developers.google.com/workspace/drive/api
- Gmail API: https://developers.google.com/workspace/gmail/api

## 5. Configure the Google Sheet

The workflows expect a sheet named `Contacts`, unless the nodes are changed.

The cleanup workflow uses this exact column name:

`Email (exact as supplied)`

Example columns:

| Column | Purpose |
|---|---|
| `Company` | Company name |
| `Location` | City or region |
| `Email (exact as supplied)` | Recruiter email |
| `send_status` | Sending status |
| `Fit` | Relevance to the candidate profile |
| `Source / context` | Source/context |
| `last_sent_at` | Last sending date |
| `send_count` | Number of sends |
| `last_error` | Error information |

Do not rename `Email (exact as supplied)` unless you also update the Code node and mappings.

If using another spreadsheet, update the document and sheet selection in every relevant Google Sheets node, including `Get row(s) in sheet` and `Delete rows or columns from sheet`.

## 6. Configure the CV in Google Drive

1. Upload the CV to Google Drive.
2. Confirm that the Google Drive credential can access it.
3. Open the `Download file` node in Workflow 1.
4. Select the correct file or replace the file ID.
5. Execute the node and verify that it returns the expected binary file.
6. Ensure the Gmail send node uses the correct binary property as its attachment.

The CV itself is not included in the workflow JSON. Each installation must configure its own file.

## 7. Configure Telegram notifications (optional)

1. Open Telegram and find `@BotFather`.
2. Send `/newbot` and follow the prompts.
3. Store the bot token securely.
4. Start a conversation with the bot and send a message.
5. Obtain the chat ID using a trusted Telegram Bot API method. If using `getUpdates`, remember that polling and a configured webhook cannot be used at the same time.
6. In n8n, create a Telegram credential and enter the bot token.
7. Select that credential in the Telegram node and configure the chat ID.
8. Send a test notification.

Never publish the bot token or a URL containing it.

## 8. Import the workflows

For each workflow JSON file:

1. Open n8n.
2. Import the workflow JSON using the workflow menu.
3. Open the workflow and inspect its nodes.
4. Select the correct local credentials in each Google Sheets, Google Drive, Gmail, or Telegram node.
5. Replace spreadsheet IDs, sheet selections, CV file IDs, chat IDs, or other account-specific values as needed.
6. Save the workflow.

Workflow JSON exports generally include node configuration and connections, not the actual OAuth tokens or API keys.

## 9. Workflow 2 — Clean failed-delivery emails

### Current flow

```text
Manual Trigger
      |
      v
Get many messages (Gmail)
      |
      +---------------------------> Delete a message (Gmail)
      |
      +--> EXTRACT FAILED EMAILS
      |             |
      |             v
      +--> Get row(s) in sheet
                    |
                    v
          FIND FAILED EMAILS IN SHEET
                    |
                    v
             Code in JavaScript
                    |
                    v
        Delete rows or columns from sheet
```

### Gmail search

The `Get many messages` node uses:

```text
from:(mailer-daemon) OR from:(postmaster)
```

There is no date restriction, but the node's configured result limit still applies. The Gmail Delete node uses the message ID expression:

```javascript
{{ $json.id }}
```

### Matching and deletion

The `EXTRACT FAILED EMAILS` Code node extracts email-shaped strings from each message's `snippet`. The matching Code node compares these addresses case-insensitively against `Email (exact as supplied)`. Matching rows are passed to the Google Sheets delete operation using `row_number`.

### Safety checks before a full cleanup

- Inspect the Gmail result count. If it reaches the configured limit, more messages may remain.
- Review extracted addresses and matching spreadsheet rows before deletion.
- The current code reads snippets only; it may miss recipients shown only in the full message body.
- A bounce message can mention multiple addresses, and the current regex may extract an address that is not the failed recipient.
- The current logic does not reliably distinguish permanent failures (for example, an address that does not exist) from temporary failures. A temporary bounce could therefore cause a valid contact to be matched.
- The Gmail deletion branch deletes all messages returned by the Gmail search, not only messages that matched a spreadsheet row.
- Gmail's Delete operation normally moves messages to Trash.
- When deleting multiple sheet rows, verify that the operation handles row positions safely. Deleting from the largest row number to the smallest can avoid row-index shifts.

For production use, filter confirmed permanent failures, preview matched rows, and only then enable automatic deletion.

## 10. Test before enabling automation

1. Read contacts from Google Sheets.
2. Download the CV.
3. Send a test email to an address you control.
4. Confirm the correct spreadsheet row is updated.
5. Test Telegram notifications, if enabled.
6. Run Workflow 2 manually and inspect the Gmail results.
7. Inspect extracted addresses and matched rows before deleting.
8. Confirm the intended Gmail messages are moved to Trash.
9. Confirm only intended spreadsheet rows are removed.

Keep workflows manual until the tests pass. Do not activate automatic bulk sending or deletion until the behavior is verified.

## 11. Security and sharing

- Do not share your personal `n8n_data` directory; it may contain sensitive instance configuration and credentials.
- Do not commit OAuth secrets, API keys, Telegram tokens, passwords, or credential exports to Git.
- Each person should configure their own credentials.
- Review workflow JSON exports for personal addresses, IDs, test data, or other private values before sharing.
- Restrict access to the spreadsheet and CV.
- Do not expose n8n to the public internet without suitable authentication, HTTPS, and network protections.
- Back up workflow JSON files and important spreadsheet data.

## 12. Final checklist

- [ ] Docker Compose starts n8n.
- [ ] Google Sheets credential is connected.
- [ ] Google Drive credential is connected.
- [ ] Gmail credential is connected.
- [ ] Spreadsheet and `Contacts` sheet are correct.
- [ ] `Email (exact as supplied)` is unchanged.
- [ ] CV file is accessible.
- [ ] All imported nodes use valid credentials.
- [ ] Workflow 1 passes a controlled test.
- [ ] Workflow 2 has been tested on a small set before deletion.
- [ ] Optional Telegram notifications work.
- [ ] Automatic execution is enabled only after successful testing.