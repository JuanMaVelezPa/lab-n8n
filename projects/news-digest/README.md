# News digest

A daily email with each user's latest news, read from RSS feeds and summarised with local Ollama.
Users, topics, feeds and settings live in a Google Sheet. The full design is in [SPEC 01](../../specs/01-daily-news-digest.md), and the email layout in [SPEC 04](../../specs/04-responsive-email-layout.md).

## How it works

The workflow has six groups of nodes, marked with sticky notes:

1. **Start.** It runs by hand or every hour. The hourly run continues only at the hour and days set in the `Config` tab.
2. **Read the sheet.** Users, their topics and the list of feeds.
3. **Pick the news.** Each feed is read once. Each user gets the newest articles of their topics, up to `news_count`.
4. **Summarise with AI.** Ollama rewrites one article at a time in the user's language and level. If a call fails, the original text stays.
5. **Vocabulary.** For users who read in English, Ollama picks 5 difficult words from the summaries and gives their Spanish meaning.
6. **Email.** One HTML email per user, sent with Gmail. Users with no articles get no email. The email is one column, up to 760px wide on a laptop and full width on a phone, with less side padding on small screens.

## Setup

You need:

- n8n running at http://localhost:5678 (see the [shared setup](../../docs/setup.md)).
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

### 2. Create the Google credentials

This project needs the `Google Sheets account` and `Gmail account` credentials in n8n.
If they do not exist yet, create them with steps 5 and 6 of the [shared setup](../../docs/setup.md#5-create-the-google-oauth-client).

Check: in n8n, open **Credentials** and confirm that both show **Account connected**.

### 3. Import the workflow

Run these from the repo root:

```bash
docker cp projects/news-digest/workflows/news-digest.json n8n:/tmp/news-digest.json
docker exec n8n n8n import:workflow --input=/tmp/news-digest.json
```

Then open the `News Digest` workflow in n8n and finish it by hand:

1. Open the `Settings` node and paste the document ID of your sheet into `sheetId`.
2. Open `Config`, `Users`, `Topics` and `Feeds`, and select the `Google Sheets account` credential in each.
3. Open `Send Email` and select the `Gmail account` credential.
4. Save.

Every import unpublishes the workflow and drops the sheet ID and the selected credentials, so these four steps are needed after each one.
See [Import a workflow](../../docs/operations.md#import-a-workflow) for the details, and [Export a workflow to the repo](../../docs/operations.md#export-a-workflow-to-the-repo) to bring a change made in the editor back into the repo.

### 4. Test and activate

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

## When something fails

| What fails | What happens |
| --- | --- |
| One feed, or a feed that changed its address | The feed is skipped. The email is sent with the articles of the other feeds. |
| Ollama, or one summary | That article keeps its original title and text, in the language of the feed. The other articles are summarised as usual. |
| The vocabulary call | The email has the summaries and no vocabulary box. |
| No fresh articles for a user | That user gets no email that day. The run does not fail. |

Errors are visible in the **Executions** list of n8n.
