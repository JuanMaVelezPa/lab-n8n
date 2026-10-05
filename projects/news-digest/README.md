# News digest

A daily email with each user's latest news, read from RSS feeds and summarised with local Ollama.
Users, topics, feeds and settings live in a Google Sheet. The full design is in [SPEC 01](../../specs/01-daily-news-digest.md).

## Setup

The Google part is done by hand, once. OAuth consent cannot be automated.

You need a Google account and n8n running at http://localhost:5678 (see the [root README](../../README.md)).

### 1. Create the Google Sheet

1. Go to https://sheets.new and name the spreadsheet `News Digest`.
2. Import each file in [`sheets/`](sheets/) as its own tab: **File → Import → Upload**, choose the CSV, then set **Import location** to **Insert new sheet(s)** and **Separator type** to **Comma**.
3. Rename the four tabs to `Users`, `Topics`, `Feeds` and `Config`. Tab names and headers are case-sensitive and must match exactly.
4. Delete the empty default tab (`Sheet1`).
5. In `Users` and `Topics`, replace `you@example.com` with the real address. The real address is typed in the sheet, not stored in this repo.
6. Copy the document ID from the URL. It is the part between `/d/` and `/edit`:

   ```
   https://docs.google.com/spreadsheets/d/<document ID>/edit
   ```

   You will paste it into the `Settings` node of the workflow.

### 2. Enable the Google APIs

1. Open https://console.cloud.google.com and create a project, for example `n8n`.
2. With that project selected, go to **APIs & Services → Library**.
3. Enable **Google Sheets API**.
4. Enable **Gmail API**.

### 3. Create the OAuth client

1. Go to **APIs & Services → OAuth consent screen** (Google Auth Platform) and click **Get started**.
2. Fill in an app name (for example `n8n`) and your email as support email.
3. Choose **External** as audience, add your email as contact, and finish.
4. In **Audience**, add under **Test users** the Google account that owns the sheet and will send the emails.
5. In **Clients**, click **Create client**:
   - **Application type:** Web application.
   - **Authorized redirect URIs:** `http://localhost:5678/rest/oauth2-credential/callback`
6. Copy the **Client ID** and the **Client secret**.

While the app is in **Testing** status, Google expires the authorisation after 7 days and both credentials must be reconnected.
To avoid this, open **Audience** and click **Publish app**. An unverified app shows a warning screen on sign-in, which is fine for personal use.

### 4. Create the credentials in n8n

Both credentials use the same Client ID and Client secret.

1. In n8n, open **Credentials → Create credential**.
2. Choose **Google Sheets OAuth2 API**, name it `Google Sheets OAuth2`, paste the Client ID and Client secret, click **Sign in with Google** and accept the permissions.
3. Repeat with **Gmail OAuth2 API**, named `Gmail OAuth2`.

Sign in with the account you added as test user. Open n8n at `http://localhost:5678`, not at another host name or IP, or the redirect URI will not match.

Check: both credentials show **Account connected**.

Credentials are never stored in this repo. n8n keeps them encrypted in the `n8n_data` volume.
