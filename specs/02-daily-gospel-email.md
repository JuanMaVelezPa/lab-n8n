# SPEC 02 — Daily gospel reflection by email

> **Status:** Implemented
> **Depends on:** none
> **Date:** 2026-10-13
> **Objective:** An n8n workflow that fetches the Catholic gospel of the day in English and Spanish, has local Ollama explain and apply it as a short question-and-answer dialogue for each group's language, level and audience, and sends one HTML email per group of recipients listed in a Google Sheet through Gmail.

## Why this spec exists

The goal is to understand the gospel of each day in an easy way and apply it to real life.
Two books set the style.
Neale Donald Walsch's "Conversations with God" gives the question-and-answer form.
"Daily Prayers for Men" gives the short, direct prayer.
The email is also English practice at a chosen level, as in SPEC 01.

This is a new project, separate from `projects/news-digest/`.
It reuses the patterns of SPEC 01 and none of its files.

## Scope

**In:**

- One workflow file, `projects/daily-gospel/workflows/daily-gospel.json`, imported into the `n8n` container with the n8n CLI.
- One new Google Sheet named `Daily Gospel`, separate from the `News Digest` sheet, with two tabs: `Users` and `Config`.
- Two CSV templates in `projects/daily-gospel/sheets/` with the exact headers and starter rows, to import as those tabs.
- Recipient groups: each row in `Users` holds one configuration and one or more email addresses.
- One Gmail message per row where `active` is `TRUE`, with all the addresses of the row in BCC.
- Per-row settings: language (`en` / `es` / `both`), English level (`B1` / `B2` / `C1` in the template), audience (`general` / `men` / `women` in the template).
- Allowed values of the three settings listed in `Config`: `languages`, `levels` and `audiences`.
- Gospel of the day from the Evangelizo feed, in English and Spanish: liturgical day, reference and full text.
- AI reflection with local Ollama (`gemma4:12b`), generated once per distinct variant (language, level, audience) and shared by all rows of that variant.
- Reflection sections: 3 question-and-answer pairs, application with one action for today, short prayer, reflection question, phrase to remember.
- Vocabulary box: 5 difficult words with their Spanish meaning, in the English part of the email.
- HTML email with inline CSS: header with the liturgical day, gospel text with reference, the reflection sections, vocabulary box, footer.
- For `language` = `both`, one email with the full English block first and the full Spanish block below.
- Global schedule from `Config`: `send_hour` and `send_days`, checked by an hourly trigger.
- A Manual Trigger that skips the schedule check, for testing.
- Sending through the Gmail node.
- Degrade and continue on failures (see Decisions).
- `projects/daily-gospel/README.md` with setup steps: sheet creation, credentials, CLI import, activation.
- A new row for this project in the Projects table of the root `README.md`.

**Out of scope (for future specs):**

- The other readings of the Mass (first reading, psalm, second reading).
- Saint of the day and the commentary that Evangelizo publishes.
- Per-user send hour and days.
- Greeting by name or any other per-person text.
- Failure alert email to the admin.
- Archive of past reflections or a sent log.
- Audio version of the email.
- Unsubscribe link or any web form to manage users.
- Cloud LLM providers.
- A reading plan outside the liturgical calendar.

## Data model

Google Sheet named `Daily Gospel`. Tab names and headers are case-sensitive and must match exactly.

**Tab `Users`** — one row per configuration, with every person who shares it.

```csv
emails,active,language,level,audience
"you@example.com, friend@example.com",TRUE,both,B2,men
```

- `emails`: one or more addresses separated by commas. Spaces around the commas are ignored.
- `active`: `TRUE` or `FALSE`. Rows with `FALSE` are ignored. It applies to the whole row.
- `language`: one of the values in `languages` of `Config`. In the template: `en`, `es` or `both`.
- `level`: one of the values in `levels` of `Config`. In the template: `B1`, `B2` or `C1`. Used only for the English block. Ignored when `language` = `es`.
- `audience`: one of the values in `audiences` of `Config`. In the template: `general`, `men` or `women`. Sets who the application and the prayer speak to.
- A row with a value outside these lists is skipped. Upper and lower case are treated as equal.
- To pause one person, remove the address from `emails`.
- Two rows with the same `language`, `level` and `audience` are allowed and still share one reflection.

**Tab `Config`** — key/value pairs.

```csv
key,value
send_hour,5
send_days,all
ollama_model,gemma4:12b
languages,"en,es,both"
levels,"B1,B2,C1"
audiences,"general,men,women"
```

- `send_hour`: 0–23, in `America/Bogota`.
- `send_days`: `all`, or a comma list of `mon,tue,wed,thu,fri,sat,sun`.
- `languages`: comma list of the languages a row may use. It can only contain `en`, `es` and `both`. Other values are ignored, because each language needs its feed code and its labels in the workflow.
- `levels`: comma list of the English levels a row may use. Open list: a new value such as `A2` is passed to the prompt as it is.
- `audiences`: comma list of the audiences a row may use. Open list: a new value such as `youth` is passed to the prompt as it is.
- A list that is missing or empty falls back to the value shown above.

**Workflow constants** — a `Settings` Set node at the start of the workflow:

```json
{
  "sheetId": "<Google Sheet document ID>",
  "ollamaUrl": "http://ollama:11434",
  "gospelFeedUrl": "https://feed.evangelizo.org/v2/reader.php"
}
```

**Evangelizo request** — `GET` on `gospelFeedUrl`, no API key. Checked on 2026-10-13 for both languages.

| Parameter | Value |
| --- | --- |
| `date` | Today in `America/Bogota`, as `YYYYMMDD`. |
| `lang` | `AM` for English, `SP` for Spanish. |
| `type` | `reading` (text), `reading_st` (short reference), `liturgic_t` (liturgical day). |
| `content` | `GSP`. Sent with `reading` and `reading_st`. |

Three calls per language, six per run.
The response is an HTML fragment, not JSON.
`reading` ends with a copyright line and a subscription line after a double `<br />`.

**Gospel object** — one per language, after cleaning:

```json
{
  "language": "en",
  "liturgicalDay": "Tuesday of the Twenty-eighth week in Ordinary Time",
  "reference": "Luke 11,37-41",
  "text": "After Jesus had spoken, a Pharisee invited him to dine at his home. ...",
  "attribution": "Copyright © Confraternity of Christian Doctrine, USCCB",
  "ok": true
}
```

**Variant** — one per distinct combination needed by the active rows that day:

```json
{ "key": "en|B2|men", "language": "en", "level": "B2", "audience": "men" }
```

- Spanish variants use `-` as level: `es|-|women`.
- A row with `language` = `both` needs two variants, for example `en|B2|men` and `es|-|men`.
- Maximum per run with the template lists: 9 English variants and 3 Spanish variants. Longer `levels` and `audiences` lists raise it.

**Ollama responses** — requested with `stream: false`, `think: false` and a JSON schema of the expected answer in `format`, on `POST /api/generate`.

Reflection, one call per variant:

```json
{
  "dialogue": [
    { "question": "...", "answer": "2-3 sentences" },
    { "question": "...", "answer": "2-3 sentences" },
    { "question": "...", "answer": "2-3 sentences" }
  ],
  "application": "one paragraph",
  "action": "one concrete action for today",
  "prayer": "3-5 lines, first person",
  "reflection_question": "...",
  "phrase": "one line to remember"
}
```

Vocabulary, one call per English variant:

```json
{ "vocabulary": [ { "word": "...", "meaning_es": "...", "example": "..." } ] }
```

Conventions:

- Cleaning of `reading`: split at the first double `<br />`, turn the remaining `<br />` into paragraphs, decode HTML entities, strip all other tags.
- `attribution` is the first line after the double `<br />`. The subscription line and its link are dropped.
- `reference` is `reading_st` with tags stripped and the trailing dot removed.
- `ok` is `false` when the call fails or the cleaned `text` is empty.
- The reflection prompt receives only the cleaned gospel text, the reference, the language, the level and the audience.
- A reflection is valid only if `dialogue` has exactly 3 pairs and the other five fields are non-empty strings. Otherwise it counts as a failed call.
- Vocabulary words are taken from the English gospel text and the English reflection of that variant.
- Recipient list of a row: `emails` split by comma, trimmed, lower-cased, empty and repeated values dropped, values without `@` dropped.
- The message has the sending Gmail account in `To` and the recipient list in `BCC`.
- Email subject: `Gospel of the day — <reference>` for `en`, `Evangelio del día — <reference>` for `es`, `Gospel of the day / Evangelio del día — <English reference>` for `both`.

## Implementation plan

1. Create `projects/daily-gospel/sheets/Users.csv` and `projects/daily-gospel/sheets/Config.csv` with the headers and starter rows above. Check: each file opens as a table with the right columns.
2. Create `projects/daily-gospel/README.md` with the sheet setup: create `Daily Gospel` from the CSVs and reuse the `Google Sheets OAuth2` and `Gmail OAuth2` credentials that already exist in n8n. Check: the new sheet is readable with the existing credential.
3. Create `projects/daily-gospel/workflows/daily-gospel.json` with Manual Trigger, the `Settings` node and two Google Sheets read nodes (`Users`, `Config`). Import it with `n8n import:workflow` inside the container. Check: a manual run shows the rows of both tabs.
4. Add the Schedule Trigger (every hour, minute 0) and a Code node that compares the current hour and weekday in `America/Bogota` with `send_hour` and `send_days`. The manual path skips this check. Check: with `send_hour` set to another hour, the scheduled path ends with no items.
5. Add a Code node that lists the languages and the distinct variants needed by the active rows, and the cleaned recipient list of each row. Check: the starter row (`both`, `B2`, `men`) produces two recipients, languages `en`, `es` and variants `en|B2|men`, `es|-|men`.
6. Add the Evangelizo HTTP Request calls for each needed language with "continue on error", and a Code node that builds the Gospel object. Check: the output has `liturgicalDay`, `reference`, `text` and `attribution` for both languages, with no HTML tags or entities.
7. Add the reflection Ollama call per variant (HTTP Request, timeout 600 s, continue on error) and a Code node that validates the JSON and sets `aiOk`. The prompt fixes the question-and-answer form, the language, the level and the audience, and forbids quoting verses that are not in the given text. Check: `en|B2|men` returns 3 pairs, application, action, prayer, question and phrase in English.
8. Add the vocabulary Ollama call, executed only for English variants whose reflection is valid. Check: the output has 5 entries with `word`, `meaning_es` and `example`.
9. Add a Code node that renders the HTML email per row from the Gospel objects, the row's variants and the vocabulary, with labels in the block's language. Check: copying the output HTML into a browser shows the full email, with two blocks for a `both` row.
10. Add the Gmail send node, one message per row with the recipient list in BCC, skipping rows with no valid address and rows whose gospel is not `ok` in any of their languages. Check: a manual run delivers the email to both addresses of the starter row.
11. Complete `projects/daily-gospel/README.md` with import, activation, how to add a person or a new configuration row, the allowed values of each column and how to change the schedule. Add the project row to the root `README.md`. Re-export the final workflow to `projects/daily-gospel/workflows/daily-gospel.json`.

## Acceptance criteria

- [x] `projects/daily-gospel/sheets/` contains `Users.csv` and `Config.csv`, and their headers match the Data model exactly.
- [x] `docker exec n8n n8n import:workflow --input=<file>` imports `projects/daily-gospel/workflows/daily-gospel.json` without errors.
- [x] A manual run sends exactly one Gmail message per row with `active` = `TRUE` and none for rows with `FALSE`.
- [x] Every address in the `emails` cell of an active row receives the email.
- [x] A recipient cannot see the other addresses of the row in the received message.
- [x] A row whose `emails` cell is empty sends nothing, and the run does not fail.
- [x] The email shows the liturgical day, the reference and the gospel text of today's date in `America/Bogota`, equal to the text on dailygospel.org for that day.
- [x] The gospel text in the email contains no HTML tags, no HTML entities and no subscription link.
- [x] The attribution line of the gospel text appears under it.
- [x] The email contains exactly 3 question-and-answer pairs, one application paragraph, one action, one prayer, one reflection question and one phrase.
- [x] For a row with `language` = `en`, every section is in English and the email has a vocabulary box of 5 words with Spanish meanings.
- [x] For a row with `language` = `es`, every section and every label is in Spanish, and there is no vocabulary box.
- [x] For a row with `language` = `both`, the email has the English block first and the Spanish block below, and the vocabulary box is in the English block.
- [x] Two rows with the same `language`, `level` and `audience` receive the same reflection text, and the executions list shows one reflection call for that variant.
- [x] A row with `audience` = `men` and a row with `audience` = `women` receive different application and prayer texts on the same day.
- [x] Changing `send_hour` in `Config` changes the hour of the next scheduled email without editing the workflow.
- [x] With `send_days` = `mon`, a scheduled run on any other day sends nothing.
- [x] With the `ollama` container stopped, the email is still sent with the liturgical day, reference, gospel text and attribution, and without reflection or vocabulary.
- [x] With `gospelFeedUrl` replaced by an invalid URL, the run finishes without error and sends no email.
- [x] A row whose `language`, `level` or `audience` is not in the lists of `Config` sends nothing, and the run does not fail.
- [x] Adding a value to `levels` or `audiences` in `Config` makes rows with that value valid without editing the workflow.
- [x] The footer states that the reflection was written by an AI.
- [x] The email renders as a single readable column in Gmail web and in the Gmail mobile app.
- [x] The root `README.md` lists `projects/daily-gospel/` with a link to this spec.

## Decisions

- **Yes:** the gospel of the Catholic daily Mass. One fixed passage per day, chosen by the user.
- **No:** an own reading plan through the four gospels. Not selected.
- **Yes:** Evangelizo feed as the text source. One free source for both languages, no API key, verified on 2026-10-13.
- **No:** Vatican News, or USCCB plus a second Spanish site. Harder to parse, or two scrapers to maintain.
- **No:** the AI writing the gospel text from memory. Models invent verses.
- **Yes:** question-and-answer dialogue about the passage, in the form of Walsch's "Conversations with God". Chosen by the user.
- **No:** God answering in first person. The AI would write sentences attributed to God that are not in the gospel.
- **No:** direct talk without dialogue. Not selected.
- **Yes:** only the form of Walsch's book is used. The content comes from the day's passage.
- **Yes:** `audience` column with `general`, `men` and `women`. Requested by the user in place of a single fixed audience.
- **Yes:** the allowed values of `language`, `level` and `audience` live in `Config`. Requested by the user during implementation: a typo skips the row, and a new level or audience needs no workflow change.
- **No:** new languages from `Config`. A language needs a feed code, labels and a subject line, so `languages` can only restrict `en`, `es` and `both`. Extensible languages go in their own spec.
- **Yes:** one row per configuration with several addresses in `emails`. Requested by the user: people with the same settings share one row, one rendered email and one Gmail send.
- **No:** one row per person. More rows to maintain and one send per person for identical content.
- **Yes:** recipients in BCC. People on the list do not see each other's addresses.
- **No:** `name` column and greeting by name. One message serves the whole row, so it carries no per-person text.
- **Yes:** `language` column with `en`, `es` and `both`. `both` gives one email, English first.
- **No:** two separate emails for `both`. Not selected.
- **Yes:** `level` column and vocabulary box. The email doubles as English practice, as in SPEC 01.
- **Yes:** `level` applies only to English. Spanish is the readers' own language.
- **Yes:** one reflection per variant, shared by its rows. The content does not depend on the person, and it keeps Ollama calls low on CPU.
- **No:** one reflection per row. Same text generated many times.
- **Yes:** each language is generated on its own from that language's gospel text. A translated reflection would not match the wording of the Spanish bible text.
- **Yes:** about 3 minutes of reading. Chosen by the user over 1 and 6 minutes.
- **Yes:** local Ollama with `gemma4:12b` and `think: false`, as in SPEC 01. Free and already on the `ia` network.
- **No:** cloud LLM. Better writing, but needs a key and costs money.
- **Yes:** a JSON schema in `format`. Found during implementation: with `format: "json"` the model returned a broken structure for the Spanish reflection, and the same prompt with a schema returned a valid one.
- **No:** `format: "json"` as in SPEC 01. The shape here is nested and the model does not always follow it.
- **Yes:** 600 s timeout for the Ollama calls. Found during implementation: one English reflection took 231 s on CPU, too close to 300 s for longer gospels.
- **Yes:** separate vocabulary call. A failed vocabulary does not lose the reflection.
- **Yes:** hourly trigger plus `send_hour` and `send_days` in `Config`. Same pattern as SPEC 01.
- **No:** per-user send hour. Not selected.
- **Yes:** degrade and continue. A failed reflection sends the gospel alone, and a failed gospel feed sends nothing.
- **No:** failure alert email. Not selected. Errors are visible in the n8n executions list.
- **Yes:** a new sheet `Daily Gospel`. The user asked for a separate workbook from `News Digest`.
- **Yes:** the existing `Google Sheets OAuth2` and `Gmail OAuth2` credentials are reused. Same Google account and scopes.
- **Yes:** `projects/daily-gospel/` as its own folder with a new workflow. Follows the layout in the root `README.md`.
- **No:** extending the news-digest workflow. Different data, schedule and recipients.
- **Yes:** the attribution line of the feed is kept under the gospel text. The bible translations are copyrighted.
- **Yes:** a footer line saying the reflection was written by an AI. Readers must not take it as church teaching.
- **Yes:** the starter `Users.csv` uses a placeholder email. The real addresses are typed in the sheet, not stored in the repo.

## Risks

| Risk | Mitigation |
| --- | --- |
| `gemma4:12b` writes something doctrinally wrong or quotes a verse that does not exist. | The prompt gives only the day's text and forbids other quotations. The real gospel text is always shown above the reflection. The footer says the reflection is AI-written. |
| Many variants on CPU make the run long, up to 12 reflections plus 9 vocabulary calls with the template lists, and more with longer lists. | Calls run one after another with a 600 s timeout. One reflection takes 3 to 4 minutes on CPU. The email arrives some minutes after `send_hour`. Fewer distinct `level` and `audience` values mean fewer calls. |
| Evangelizo changes its HTML fragment or its parameters. | Cleaning is in one Code node. An empty text sets `ok` to `false` and no broken email is sent. |
| Evangelizo is down at `send_hour`. | No email that day. A manual run sends it later. |
| The feed's date differs from the user's day near midnight. | The `date` parameter is always computed in `America/Bogota`. |
| The model returns invalid or incomplete JSON. | The JSON schema in `format` forces the shape. An answer that still fails validation is treated as a failed call: that variant's rows get the gospel alone. |
| The bible translations are copyrighted and the feed is meant for personal use. | The list is small and private. The attribution line is kept. No public redistribution. |
| One mistyped address in a row with several. | Values without `@` are dropped before sending. A bounce for one address does not stop the others. |
| A typo in a header or in `language`, `level` or `audience` breaks matching silently. | CSV templates guarantee the headers. Values are checked against the lists in `Config`, and skipped rows are listed in the output of the plan node. `projects/daily-gospel/README.md` explains the lists. |
| A vague word in `audiences` gives a vague reflection. | The template lists three clear audiences. New ones are added on purpose in `Config`. |

## What is **not** in this spec

- Other Mass readings, psalm or saint of the day.
- Per-user schedule.
- Greeting by name.
- Failure alert email.
- Archive or sent log.
- Audio version.
- Unsubscribe link or user management form.
- Cloud LLM providers.
- Reading plans outside the liturgical calendar.

Each one of those, if it lands, goes in its own spec.
