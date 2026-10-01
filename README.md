# Recruitment Agent — Installation & Credentials Setup

## 1. Project Overview

This project uses n8n to automate recruitment activities:

* **Workflow 1 — Send Applications:** Reads recruiter contacts from Google Sheets, downloads a CV from Google Drive, sends applications using Gmail, and updates the spreadsheet.
* **Workflow 2 — Find New Recruiter Emails:** Searches the web for new recruitment contacts using OpenAI, checks for duplicates, and adds new contacts to Google Sheets.
* **Optional notifications:** Sends execution summaries through Telegram.
* **Optional bounce cleanup:** Detects failed email deliveries, matches invalid addresses against Google Sheets, and cleans up bounce notifications in Gmail.

Each person installing this project must configure their own credentials and provide access to their own Google files.

---

## 2. Prerequisites

Install the following software:

* Docker Desktop
* Docker Compose (included with current Docker Desktop installations)
* A Google account
* A Telegram account (if Telegram notifications are enabled)
* An OpenAI API account (if the email-discovery workflow uses OpenAI)

Verify Docker installation:

```bash
docker --version
docker compose version
```

---

## 3. Start n8n with Docker Compose

Create a project directory:

```text
recruitment-agent/
├── docker-compose.yml
├── README.md
├── workflows/
│   ├── send-applications.json
│   └── find-new-emails.json
└── n8n_data/
```

The `n8n_data` directory will be created automatically when Docker starts the container.

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

Start the application from the project directory:

```bash
docker compose up -d
```

Check the container:

```bash
docker compose ps
```

Open n8n:

http://localhost:5678

Create your own n8n owner account when prompted.

**Important:** For reproducible installations, use the same tested n8n image version as the original installation instead of relying indefinitely on `latest`.

---

# 4. Configure Google Credentials

Google credentials are required for:

* Reading recruitment contacts from Google Sheets.
* Adding new contacts.
* Updating sending status and counters.
* Deleting invalid contact rows.
* Downloading the CV from Google Drive.
* Sending applications through Gmail.

## 4.1 Create a Google Cloud project

1. Open https://console.cloud.google.com/

2. Sign in with your Google account.

3. Create a project, for example:

   `Recruitment Agent`

4. Select the newly created project.

## 4.2 Enable Google APIs

Open **APIs & Services → Library**.

Search for and enable these APIs:

* Google Sheets API
* Google Drive API
* Gmail API

Wait for each API to finish enabling.

## 4.3 Configure the OAuth consent screen

1. Open **Google Auth Platform** in Google Cloud Console.
2. Configure the application name and support email.
3. Enter a developer contact email.
4. Configure the audience according to your account type and intended users.
5. If the application is in Testing mode, add the Google account you will use as a test user.

Google may require additional verification or configuration depending on the OAuth scopes and whether the app is used by other Google accounts.

## 4.4 Create OAuth credentials

1. Open **APIs & Services → Credentials**.

2. Click **Create Credentials → OAuth client ID**.

3. Choose **Web application**.

4. Give it a name, for example:

   `n8n Local`

5. Under **Authorized redirect URIs**, add:

   ```text
   http://localhost:5678/rest/oauth2-credential/callback
   ```

6. Click **Create**.

7. Copy the **Client ID** and **Client Secret**.

Never publish the Client Secret in the README or a public repository.

**Important:** The redirect URI must match the URL used to access n8n. If n8n is hosted on another domain, use the corresponding callback URL shown by the n8n credential form.

## 4.5 Connect Google Sheets in n8n

1. Open n8n.
2. Open **Credentials → Create Credential**.
3. Search for the Google Sheets OAuth2 credential type.
4. Enter the Client ID and Client Secret from Google Cloud.
5. Save the credential.
6. Click **Connect my account** or the equivalent authorization button.
7. Sign in with the Google account containing the recruitment spreadsheet.
8. Review and authorize the requested permissions.

Name the credential:

`Google Sheets account`

If the credential form asks for a redirect URL, compare it with the URI configured in Google Cloud.

## 4.6 Connect Google Drive in n8n

Repeat the OAuth setup using the Google Drive credential type.

You can reuse the same Google OAuth client ID and secret when the Google Cloud client is configured for the required APIs and scopes.

Name the credential:

`Google Drive account`

Authorize the Google account containing the CV file.

## 4.7 Connect Gmail in n8n

1. Create a Gmail OAuth2 credential in n8n.
2. Enter the same OAuth Client ID and Client Secret, if appropriate.
3. Authorize the Gmail account used to send applications and receive delivery-failure notifications.
4. Save the credential.

Name the credential:

`Gmail account`

Make sure the authorized Google account has permission to access the intended mailbox.

### Important Google account notes

* The spreadsheet must be accessible to the connected account.
* The CV must be accessible through Google Drive.
* The Gmail account must be authorized for sending and reading messages.
* If Google OAuth is in Testing mode, refresh tokens may expire or require reauthorization.
* If you encounter `invalid_client`, verify the Client ID and Client Secret.
* If you encounter a redirect URI error, verify that the callback URL matches exactly.

Official documentation:

* Google OAuth: https://developers.google.com/identity/protocols/oauth2
* Google Sheets API: https://developers.google.com/workspace/sheets/api
* Google Drive API: https://developers.google.com/workspace/drive/api
* Gmail API: https://developers.google.com/workspace/gmail/api

---

# 5. Configure the Google Spreadsheet

Create or copy the recruitment contacts spreadsheet.

The spreadsheet should contain the sheet expected by the workflows, for example:

`Contacts`

The column names must match the expressions and mappings used by the workflows.

Example:

| Column                    | Purpose                              |
| ------------------------- | ------------------------------------ |
| Company                   | Company name                         |
| Location                  | City or region                       |
| Email (exact as supplied) | Recruiter email address              |
| send_status               | Sending status                       |
| Fit                       | Relevance to the candidate's profile |
| Source / context          | Source or recruitment context        |
| last_sent_at              | Last sending date                    |
| send_count                | Number of sending attempts           |
| last_error                | Error information                    |

**Do not rename `Email (exact as supplied)` to `email` unless you also update the relevant workflow code and mappings.**

If the spreadsheet belongs to another Google account, share it with the account used by the n8n Google Sheets credential, or copy it into that account's Drive.

Update the spreadsheet selection in each Google Sheets node if the document ID changes.

---

# 6. Configure the CV in Google Drive

The sending workflow downloads a CV before attaching it to outgoing emails.

1. Upload the CV to Google Drive.
2. Make sure the Google Drive credential can access the file.
3. Open the `Download file` node.
4. Select the new CV file or replace the existing file ID.
5. Execute the node.
6. Verify that the output contains the expected binary file.

The Gmail node must use the binary property produced by the Download file node as its attachment.

**Important:** A workflow export does not include the CV itself. Each person must provide their own CV and configure the corresponding Drive file.

---

# 7. Configure OpenAI for Email Discovery

The email-discovery workflow uses OpenAI to search for recruitment contacts and return structured results.

## 7.1 Obtain an API key

1. Open https://platform.openai.com/
2. Sign in or create an account.
3. Open the API key management page.
4. Create a new API key.
5. Copy it securely.

Do not put the API key in the workflow's Code node, README, or Git repository.

API usage may require an available balance, billing configuration, or an eligible API plan. ChatGPT subscriptions and API billing are separate.

## 7.2 Add the credential in n8n

1. Open **Credentials → Create Credential**.
2. Search for **OpenAI**.
3. Enter the API key.
4. Save the credential.

Name it:

`OpenAI account`

## 7.3 Configure the OpenAI node

1. Open the email-discovery workflow.
2. Open the `Message a Model` or equivalent OpenAI node.
3. Select the new OpenAI credential.
4. Select a model available to the account.
5. Enable the built-in Web Search tool if the workflow uses it.
6. Preserve the prompt and expected output structure.
7. Execute a test search.

Verify that the response contains valid company names, email addresses, and source URLs. Do not assume that an address is valid merely because an AI model returned it.

Official documentation:

* API keys: https://platform.openai.com/api-keys
* OpenAI API: https://platform.openai.com/docs
* n8n OpenAI integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/

---

# 8. Configure Telegram Notifications (Optional)

Telegram is used to send execution summaries.

## 8.1 Create a bot

1. Open Telegram.
2. Search for `@BotFather`.
3. Send `/newbot`.
4. Choose a display name.
5. Choose a unique username ending in `bot`.
6. Copy the bot token provided by BotFather.

Keep the token private.

## 8.2 Obtain the chat ID

1. Open the newly created bot using its Telegram link.
2. Press **Start** and send a message.
3. Retrieve the chat ID using a trusted Telegram bot-management method or the Telegram Bot API.
4. If using `getUpdates`, ensure a webhook is not currently configured for the bot because polling and webhooks are mutually exclusive.

Do not publish the bot token or a URL containing the token.

## 8.3 Configure n8n

1. Open **Credentials → Create Credential**.
2. Select the Telegram credential type.
3. Enter the bot token.
4. Save the credential.
5. Open each Telegram node.
6. Select the new credential.
7. Enter the correct chat ID.
8. Execute a test notification.

If the workflow sends notifications to a group, add the bot to that group and use the appropriate group chat ID.

---

# 9. Import the Workflows

Repeat these steps for each exported workflow.

1. Open n8n.
2. Use the workflow menu to import the JSON file.
3. Open the imported workflow.
4. Inspect each Google Sheets, Google Drive, Gmail, OpenAI, and Telegram node.
5. Select the appropriate local credential for each node.
6. Replace spreadsheet, Drive file, and other account-specific identifiers as necessary.
7. Save the workflow.

An exported workflow normally preserves its nodes, connections, expressions, and configuration, but it does not transfer the actual OAuth secrets or API keys.

---

# 10. Test Before Running Automatically

Test the workflows in this order:

1. **Google Sheets:** Verify that contacts can be read.
2. **Google Drive:** Verify that the CV downloads successfully.
3. **Gmail:** Send a test email to an address you control.
4. **Google Sheets update:** Verify that the correct row is updated.
5. **OpenAI:** Verify that a test search returns structured contacts and source URLs.
6. **Duplicate detection:** Confirm that existing email addresses are excluded.
7. **Telegram:** Verify that a test notification arrives.
8. **Bounce cleanup:** Verify that only the intended failed addresses are selected before deleting spreadsheet rows.

Start with manual execution and a small test dataset.

Do not activate automatic sending until all tests pass. Ensure that the workflow cannot send duplicate applications or delete valid contact rows.

---

# 11. Security Rules

* Never commit API keys, OAuth client secrets, Telegram tokens, passwords, or exported credential data.
* Do not share your personal `n8n_data` directory with another person.
* Use separate credentials and accounts for each installation.
* Restrict access to the spreadsheet and CV.
* Do not expose the n8n editor directly to the public internet without appropriate authentication, HTTPS, and network protections.
* Back up workflow JSON files and important spreadsheet data.
* Review all recipients before enabling bulk email sending.

---

# 12. Final Checklist

* [ ] Docker Desktop is installed.
* [ ] n8n starts successfully.
* [ ] Google Sheets credential is connected.
* [ ] Google Drive credential is connected.
* [ ] Gmail credential is connected.
* [ ] OpenAI credential is connected, if required.
* [ ] Telegram credential is connected, if required.
* [ ] Spreadsheet and sheet names are correct.
* [ ] CV file is accessible.
* [ ] All imported nodes reference valid credentials.
* [ ] Duplicate detection works.
* [ ] Test email and spreadsheet updates work.
* [ ] Bounce cleanup has been tested safely.
* [ ] Automatic workflows are activated only after successful tests.

**Installation complete:** The recruitment agent is ready for testing on the new machine.
