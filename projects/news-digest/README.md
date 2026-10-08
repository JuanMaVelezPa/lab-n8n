# News digest

A daily email with each user's latest news, read from RSS feeds and summarised with local Ollama.
Users, topics, feeds and settings live in a Google Sheet. The full design is in [SPEC 01](../../specs/01-daily-news-digest.md).

## How it works

The workflow has six groups of nodes, marked with sticky notes:

1. **Start.** It runs by hand or every hour. The hourly run continues only at the hour and days set in the `Config` tab.
2. **Read the sheet.** Users, their topics and the list of feeds.
3. **Pick the news.** Each feed is read once. Each user gets the newest articles of their topics, up to `news_count`.
4. **Summarise with AI.** Ollama rewrites one article at a time in the user's language and level. If a call fails, the original text stays.
5. **Vocabulary.** For users who read in English, Ollama picks 5 difficult words from the summaries and gives their Spanish meaning.
6. **Email.** One HTML email per user, sent with Gmail. Users with no articles get no email.

## Setup

The Google part is done by hand, once. OAuth consent cannot be automated.

You need:

- A Google account.
- n8n running at http://localhost:5678 (see the [root README](../../README.md)).
- The `ollama` container on the same Docker network, with the model from the `Config` tab pulled (`gemma4:12b` in the template).

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

### 5. Import the workflow

Run these from the repo root. The file is copied into the container first, because the container cannot see this folder.

```bash
docker cp projects/news-digest/workflows/news-digest.json n8n:/tmp/news-digest.json
docker exec n8n n8n import:workflow --input=/tmp/news-digest.json
```

Then open the `News Digest` workflow in n8n and finish it by hand:

1. Open the `Settings` node and paste the document ID of your sheet into `sheetId`.
2. Open `Config`, `Users`, `Topics` and `Feeds`, and select the `Google Sheets OAuth2` credential in each.
3. Open `Send Email` and select the `Gmail OAuth2` credential.
4. Save.

Importing again replaces the workflow in n8n with the file. The sheet ID and the selected credentials are lost and these four steps must be repeated.

### 6. Test and activate

1. Click **Execute workflow**. The manual run skips the schedule check and sends the emails now.
2. Wait for it to finish. On CPU, Ollama needs about 25 seconds per article and about 80 seconds for the vocabulary box.
3. When the email looks right, click **Publish** (the **Active** switch in older n8n versions). From then on the workflow wakes up every hour and sends only at the hour and days set in `Config`.

If a user gets no email, open the last run in **Executions** and look at `Build Plan`. It shows the topics and the number of matching feeds for each user.

## Daily use

Everything below is done in the Google Sheet. The workflow does not change.

Values must be typed exactly as shown. A typo does not raise an error: the row simply matches nothing.

### Add a user

1. Add a row to `Users`:

   | Column | Allowed values |
   | --- | --- |
   | `email` | The address that receives the email. |
   | `name` | Shown in the greeting. |
   | `active` | `TRUE` or `FALSE`. Rows with `FALSE` get no email. |
   | `language` | `en` or `es`. Language of the summaries and of the email labels. |
   | `level` | `B1`, `B2` or `C1`. Used only when `language` is `en`. |
   | `scope` | `national` (Colombia), `international` or `both`. |
   | `news_count` | Total number of articles in the email, from 1 to 15. |

2. Add one row to `Topics` for each topic the user wants.

A user with no rows in `Topics` gets no email.

### Add a topic to a user

Add a row to `Topics` with the user's email and the topic. The topic must be one of the words used in the `topic` column of `Feeds`.

The feeds in the template use these topics:

| Topic | Feeds |
| --- | --- |
| `technology` | BBC Technology, El Tiempo Tecnosfera |
| `economy` | BBC Business, El Tiempo Economía |
| `politics` | BBC World, El Tiempo Política |
| `sports` | BBC Sport, El Tiempo Deportes |
| `health` | BBC Health, El Tiempo Salud |
| `science` | BBC Science, El Tiempo Ciencia |
| `culture` | BBC Culture, El Tiempo Cultura |

### Add a feed

Add a row to `Feeds`:

| Column | Allowed values |
| --- | --- |
| `name` | Shown in the email as the source. |
| `url` | The address of the RSS feed. |
| `topic` | An existing topic, or a new word to create a topic. |
| `scope` | `national` (Colombia) or `international`. |
| `language` | `en` or `es`. Language the feed is written in. |
| `active` | `TRUE` or `FALSE`. Rows with `FALSE` are not read. |

A new topic works as soon as a user has it in `Topics`. In emails in Spanish, a topic that is not in the table above is shown as it is typed in the sheet.

A feed that fails or changes its address is skipped, and the email is sent with the other feeds.

### Change the schedule

Edit the `Config` tab:

| Key | Allowed values |
| --- | --- |
| `send_hour` | 0 to 23, in Colombia time (`America/Bogota`). |
| `send_days` | `all`, or a list such as `mon,tue,wed,thu,fri`. Days are `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun`. |
| `ollama_model` | The Ollama model that writes the summaries. It must be pulled in the `ollama` container. |
| `max_article_age_hours` | Articles older than this are left out. A higher value gives more articles, but they can repeat the next day. |

The change applies from the next hour. The email arrives some minutes after `send_hour`, because the summaries are written first.
