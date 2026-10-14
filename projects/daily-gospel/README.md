# Daily gospel

A daily email with the Catholic gospel of the day and a short reflection written with local Ollama.
On the days the passage is hard to understand, it also has a two-line apologetic note.
Recipients and settings live in a Google Sheet. The full design is in [SPEC 02](../../specs/02-daily-gospel-email.md) and [SPEC 03](../../specs/03-gospel-apologetic-note.md), and the email layout in [SPEC 04](../../specs/04-responsive-email-layout.md).

## How it works

The workflow has seven groups of nodes, marked with sticky notes:

1. **Start.** It runs by hand or every hour. The hourly run continues only at the hour and days set in the `Config` tab.
2. **Read the sheet.** Each active row of `Users` is one configuration with its addresses. The workflow lists the reflections it needs: one per distinct language, level and audience.
3. **Gospel of the day.** The text, the reference and the liturgical day come from the [Evangelizo](https://evangelizo.org) feed, in English and Spanish.
4. **Apologetic note.** Ollama decides, once per language, whether the passage has a custom, an image, an expression or a saying that a modern reader would misread. Only then it writes two lines: the context and the meaning. The email shows them after the gospel text, under `Apologetic note` or `Nota apologética`.
5. **Reflection with AI.** Ollama writes three questions and answers about the passage, an application with one action for today, a short prayer, a question and a phrase to remember. It receives only the text of the day and, when there is one, the apologetic note.
6. **Vocabulary.** For English, Ollama picks 5 difficult words from the gospel and the reflection and gives their Spanish meaning.
7. **Email.** One HTML email per row, sent with Gmail to the sending account, with the addresses of the row in BCC. The email is one column, up to 760px wide on a laptop and full width on a phone, with less side padding on small screens.

The gospel text is never written by the AI. The footer of the email says that the note and the reflection are AI-written and are not church teaching.

## Setup

You need:

- n8n running at http://localhost:5678 (see the [root README](../../README.md)).
- The `ollama` container on the same Docker network, with the model from the `Config` tab pulled (`gemma4:12b` in the template).
- The `Google Sheets OAuth2` and `Gmail OAuth2` credentials in n8n.

### 1. Create the Google Sheet

This project has its own spreadsheet, separate from `News Digest`.

1. Go to https://sheets.new and name the spreadsheet `Daily Gospel`. Use the same Google account as the n8n credentials.
2. Import each file in [`sheets/`](sheets/) as its own tab: **File → Import → Upload**, choose the CSV, then set **Import location** to **Insert new sheet(s)** and **Separator type** to **Comma**.
3. Rename the two tabs to `Users` and `Config`. Tab names and headers are case-sensitive and must match exactly.
4. Delete the empty default tab (`Sheet1`).
5. In `Users`, replace `you@example.com, friend@example.com` with the real addresses, separated by commas. The real addresses are typed in the sheet, not stored in this repo.
6. Copy the document ID from the URL. It is the part between `/d/` and `/edit`:

   ```
   https://docs.google.com/spreadsheets/d/<document ID>/edit
   ```

   You will paste it into the `Settings` node of the workflow.

### 2. Credentials

This project reuses the two credentials created for the news digest: `Google Sheets OAuth2` and `Gmail OAuth2`.
They work for the new sheet as they are, because it belongs to the same Google account and needs the same permissions.

If they do not exist yet, create them with steps 2 to 4 of the [news digest setup](../news-digest/README.md#2-enable-the-google-apis).

Check: in n8n, open **Credentials** and confirm that both show **Account connected**.

Credentials are never stored in this repo. n8n keeps them encrypted in the `n8n_data` volume.

### 3. Import the workflow

Run these from the repo root. The file is copied into the container first, because the container cannot see this folder.

```bash
docker cp projects/daily-gospel/workflows/daily-gospel.json n8n:/tmp/daily-gospel.json
docker exec n8n n8n import:workflow --input=/tmp/daily-gospel.json
```

Then open the `Daily Gospel` workflow in n8n and finish it by hand:

1. Open the `Settings` node and paste the document ID of your sheet into `sheetId`.
2. Open `Config` and `Users`, and select the Google Sheets credential in each.
3. Open `Sender` and `Send Email`, and select the Gmail credential in each.
4. Save.

Importing again replaces the workflow in n8n with the file. The sheet ID and the selected credentials are lost and these four steps must be repeated.
Importing also unpublishes the workflow. Click **Publish** again after the four steps.

### 4. Test and activate

1. Click **Execute workflow**. The manual run skips the schedule check and sends the emails now.
2. Wait for it to finish. On CPU, Ollama needs about 1 minute per apologetic note, 3 to 4 minutes per reflection and about 2 minutes per vocabulary box. A row with `both` takes about 9 minutes.
3. When the email looks right, click **Publish** (the **Active** switch in older n8n versions). From then on the workflow wakes up every hour and sends only at the hour and days set in `Config`.

If a row gets no email, open the last run in **Executions** and look at `Build Plan`. It shows the recipients and the variants of each row, the rows it skipped, and the allowed values it read from `Config`.

## Daily use

Everything below is done in the Google Sheet. The workflow does not change.

### Add a person

People with the same settings share one row. Add the address to the `emails` cell of that row, after a comma:

```
you@example.com, friend@example.com, new@example.com
```

To pause one person, remove the address from the cell. To pause everyone in a row, set `active` to `FALSE`.

Each row is sent as one message with all its addresses in BCC, so the people in a row do not see each other's addresses.

### Add a configuration

Add a row to `Users` when someone needs different settings:

| Column | Allowed values |
| --- | --- |
| `emails` | One or more addresses separated by commas. Values without `@` are dropped. |
| `active` | `TRUE` or `FALSE`. Rows with `FALSE` get no email. |
| `language` | `en`, `es` or `both`. With `both`, the email has the English block first and the Spanish block below. |
| `level` | `B1`, `B2` or `C1`. Level of the English text. Not used when `language` is `es`. |
| `audience` | `general`, `men` or `women`. Who the application and the prayer speak to. |

Upper and lower case are treated as equal. A row with a value that is not allowed gets no email, and no error is raised. It appears under `skipped` in the `Build Plan` node of the run.

Rows with the same `language`, `level` and `audience` share one reflection. Each new combination adds one Ollama call for the reflection, and one more for the vocabulary if it is in English.

### Change the allowed values

The values in the table above are the ones in the template. They live in the `Config` tab:

| Key | Allowed values |
| --- | --- |
| `languages` | A list with `en`, `es` and `both`, or some of them. Removing one turns it off. A new language cannot be added here, because it needs changes in the workflow. |
| `levels` | Any list of levels, such as `B1,B2,C1`. A new one, such as `A2`, works at once. |
| `audiences` | Any list of audiences, such as `general,men,women`. A new one, such as `youth`, works at once. |

A new level or audience is given to the AI as it is typed. Use a clear word.

If a key is missing or its value is empty, the template list is used.

### Turn the apologetic note on or off

Edit the `Config` tab:

| Key | Allowed values |
| --- | --- |
| `apologetics` | `on` or `off`. With `off`, the note is not written and no email shows it. |

If the key is missing, or has any other value, it counts as `on`.

With `on`, the AI still decides each day. A clear passage gets no note, and the email has no `Apologetic note` section that day.
The note is the same for every row of a language. In English it is always written in simple English, for all levels.
In an email with `both`, one block can have the note and the other not, because each language decides on its own.

To see the decision of a day, open the run in **Executions** and look at `Apply Apologetics`. It shows `needed` and `ok` for each language.

### Change the schedule

Edit the `Config` tab:

| Key | Allowed values |
| --- | --- |
| `send_hour` | 0 to 23, in Colombia time (`America/Bogota`). |
| `send_days` | `all`, or a list such as `mon,tue,wed,thu,fri`. Days are `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun`. |
| `ollama_model` | The Ollama model that writes the reflection. It must be pulled in the `ollama` container. |

The change applies from the next hour. The email arrives some minutes after `send_hour`, because the reflections are written first.

### When something fails

| What fails | What happens |
| --- | --- |
| Ollama, or one reflection | The rows of that variant get the gospel alone, with no reflection and no vocabulary. |
| The apologetic note call | The email has no `Apologetic note` section. The reflection is written without it. |
| The vocabulary call | The email has the reflection and no vocabulary box. |
| The gospel feed in one language | A row with `both` gets the block of the other language. A row with only that language gets no email. |
| The gospel feed in both languages | No email that day. Run the workflow by hand later. |

Errors are visible in the **Executions** list of n8n.
