# SPEC 01 — Daily news digest by email

> **Status:** Implemented
> **Depends on:** none
> **Date:** 2026-10-04
> **Objective:** An n8n workflow that reads users, topics, feeds and settings from a Google Sheet, picks each user's latest news from RSS feeds, summarises them with local Ollama in the user's language and level, and sends one HTML email per user through Gmail.

## Why this spec exists

This is the first n8n project in this folder.
It has two goals: a useful daily email, and English practice at B2 level.
The second goal is why the summaries are rewritten by an AI at a chosen level instead of being copied from the feed.

## Scope

**In:**

- One workflow file, `projects/news-digest/workflows/news-digest.json`, imported into the `n8n` container with the n8n CLI.
- One Google Sheet with four tabs: `Users`, `Topics`, `Feeds`, `Config`.
- Four CSV templates in `projects/news-digest/sheets/` with the exact headers and starter rows, to import as those tabs.
- Multi-user support: one email per row in `Users` where `active` is `TRUE`.
- Per-user settings: language (`en` / `es`), English level (`B1` / `B2` / `C1`), scope (`national` / `international` / `both`), number of news.
- Per-user topics, one row per topic in `Topics`.
- News from a fixed list of RSS feeds in `Feeds`, each tagged with topic, scope and language.
- Starter feeds for seven topics (`technology`, `economy`, `politics`, `sports`, `health`, `science`, `culture`), one Colombian and one international each.
- Global schedule from `Config`: `send_hour` and `send_days`, checked by an hourly trigger.
- A Manual Trigger that skips the schedule check, for testing.
- AI summary per article with local Ollama (`gemma4:12b`), written in the user's language and level, translating when the feed is in another language.
- Vocabulary box: 5 difficult words from that day's summaries with their Spanish meaning, only for users with `language` = `en`.
- HTML email with inline CSS: header, one card per article (topic, title, summary, source, link), vocabulary box, footer.
- Sending through the Gmail node.
- Degrade and continue on failures (see Decisions).
- `projects/news-digest/README.md` with setup steps: Google credentials, sheet creation, CLI import, activation.

**Out of scope (for future specs):**

- No-repeat log (`SentLog` sheet).
- Per-user send hour and days.
- Failure alert email to the admin.
- Keyword filtering or AI classification of articles by topic.
- News APIs (NewsAPI, GNews) and Google News RSS search.
- Fetching the full article body from the publisher's site.
- Unsubscribe link or any web form to manage users.
- Cloud LLM providers.

## Data model

Google Sheet named `News Digest`. Tab names and headers are case-sensitive and must match exactly.

**Tab `Users`** — one row per person.

```csv
email,name,active,language,level,scope,news_count
you@example.com,Juan,TRUE,en,B2,both,6
```

- `active`: `TRUE` or `FALSE`. Rows with `FALSE` are ignored.
- `language`: `en` or `es`. Language of summaries and of the email labels.
- `level`: `B1`, `B2` or `C1`. Used only when `language` = `en`.
- `scope`: `national`, `international` or `both`.
- `news_count`: total number of articles in the email, integer from 1 to 15.

**Tab `Topics`** — one row per user and topic.

```csv
user_email,topic
you@example.com,technology
you@example.com,economy
```

- `user_email` must match `Users.email`.
- `topic` must match a value used in `Feeds.topic`.

**Tab `Feeds`** — one row per RSS feed. The first six URLs were checked on 2026-10-04 and the other eight on 2026-10-08. All returned valid RSS.

```csv
name,url,topic,scope,language,active
BBC Technology,https://feeds.bbci.co.uk/news/technology/rss.xml,technology,international,en,TRUE
BBC Business,https://feeds.bbci.co.uk/news/business/rss.xml,economy,international,en,TRUE
BBC World,https://feeds.bbci.co.uk/news/world/rss.xml,politics,international,en,TRUE
El Tiempo Tecnosfera,https://www.eltiempo.com/rss/tecnosfera.xml,technology,national,es,TRUE
El Tiempo Economía,https://www.eltiempo.com/rss/economia.xml,economy,national,es,TRUE
El Tiempo Política,https://www.eltiempo.com/rss/politica.xml,politics,national,es,TRUE
BBC Sport,https://feeds.bbci.co.uk/sport/rss.xml,sports,international,en,TRUE
BBC Health,https://feeds.bbci.co.uk/news/health/rss.xml,health,international,en,TRUE
BBC Science,https://feeds.bbci.co.uk/news/science_and_environment/rss.xml,science,international,en,TRUE
BBC Culture,https://feeds.bbci.co.uk/news/entertainment_and_arts/rss.xml,culture,international,en,TRUE
El Tiempo Deportes,https://www.eltiempo.com/rss/deportes.xml,sports,national,es,TRUE
El Tiempo Salud,https://www.eltiempo.com/rss/salud.xml,health,national,es,TRUE
El Tiempo Ciencia,https://www.eltiempo.com/rss/vida_ciencia.xml,science,national,es,TRUE
El Tiempo Cultura,https://www.eltiempo.com/rss/cultura.xml,culture,national,es,TRUE
```

- `scope`: `national` (Colombia) or `international`.
- `name` is shown in the email as the source.
- In emails in Spanish, the seven starter topics are shown translated. Any other topic is shown as typed in the sheet.

**Tab `Config`** — key/value pairs.

```csv
key,value
send_hour,6
send_days,all
ollama_model,gemma4:12b
max_article_age_hours,24
```

- `send_hour`: 0–23, in `America/Bogota`.
- `send_days`: `all`, or a comma list of `mon,tue,wed,thu,fri,sat,sun`.
- `max_article_age_hours`: articles older than this are discarded.

**Workflow constants** — a `Settings` Set node at the start of the workflow holds the values that cannot live in the sheet:

```json
{
  "sheetId": "<Google Sheet document ID>",
  "ollamaUrl": "http://ollama:11434"
}
```

**Article object** passed between nodes:

```json
{
  "topic": "technology",
  "source": "BBC Technology",
  "link": "https://...",
  "publishedAt": "2026-10-04T10:54:36-05:00",
  "originalTitle": "...",
  "originalSnippet": "...",
  "title": "...",
  "summary": "...",
  "aiOk": true
}
```

**Ollama responses** — requested with `format: "json"`, `stream: false` and `think: false` on `POST /api/generate`:

```json
{ "title": "...", "summary": "2-3 sentences" }
```

```json
{ "vocabulary": [ { "word": "...", "meaning_es": "...", "example": "..." } ] }
```

Conventions:

- Article selection: feeds matching the user's topics and scope, items newer than `max_article_age_hours`, de-duplicated by `link`, picked round-robin across topics newest first until `news_count` is reached.
- If a topic has fewer fresh items than its share, the free slots go to the other topics.
- Each distinct feed URL is fetched once per run, even if several users need it.
- Email subject: `Your news — YYYY-MM-DD` for `en`, `Tus noticias — YYYY-MM-DD` for `es`.

## Implementation plan

1. Create `projects/news-digest/sheets/Users.csv`, `projects/news-digest/sheets/Topics.csv`, `projects/news-digest/sheets/Feeds.csv` and `projects/news-digest/sheets/Config.csv` with the headers and starter rows above. Check: each file opens as a table with the right columns.
2. Create `projects/news-digest/README.md` with the Google part of the setup: create the sheet from the CSVs, enable the Sheets and Gmail APIs, create the OAuth client, create the `Google Sheets OAuth2` and `Gmail OAuth2` credentials in n8n. Check: both credentials show "connected" in n8n.
3. Create `projects/news-digest/workflows/news-digest.json` with Manual Trigger, the `Settings` node and four Google Sheets read nodes (`Users`, `Topics`, `Feeds`, `Config`). Import it with `n8n import:workflow` inside the container. Check: a manual run shows the rows of the four tabs.
4. Add the Schedule Trigger (every hour, minute 0) and a Code node that compares the current hour and weekday in `America/Bogota` with `send_hour` and `send_days`. The manual path skips this check. Check: with `send_hour` set to another hour, the scheduled path ends with no items.
5. Add a Code node that builds, for each active user, the list of matching feeds, and the list of distinct feed URLs. Check: output shows the expected feeds for the starter user.
6. Add the RSS fetch for the distinct feeds with "continue on error", and a Code node that applies the age filter, de-duplication and round-robin selection per user. Check: the starter user gets at most `news_count` articles with the Article fields filled.
7. Add the per-article Ollama call (HTTP Request, timeout 300 s, continue on error) with a prompt that uses `language` and `level`, and a Code node that sets `title`, `summary` and `aiOk`, falling back to the original text when the call fails. Check: articles from a Spanish feed come back with an English summary for an `en` user.
8. Add the vocabulary Ollama call, executed only for users with `language` = `en`. Check: the output has 5 entries with `word`, `meaning_es` and `example`.
9. Add a Code node that renders the HTML email from the user, their articles and the vocabulary, with labels in the user's language. Check: copying the output HTML into a browser shows the full email.
10. Add the Gmail send node, one message per user, skipping users with zero articles. Check: a manual run delivers the email to the starter user.
11. Complete `projects/news-digest/README.md` with import, activation, how to add a user, a topic or a feed, and how to change the schedule. Re-export the final workflow to `projects/news-digest/workflows/news-digest.json`.

## Acceptance criteria

- [X] `projects/news-digest/sheets/` contains the four CSV files, and their headers match the Data model exactly.
- [X] `docker exec n8n n8n import:workflow --input=<file>` imports `projects/news-digest/workflows/news-digest.json` without errors.
- [X] A manual run sends one email to each user with `active` = `TRUE` and none to users with `FALSE`.
- [X] The email contains at most `news_count` articles, and no link appears twice.
- [X] Every article shows topic, title, summary, source name and a working link to the original.
- [X] For a user with `language` = `en`, all summaries are in English, including those from El Tiempo feeds.
- [X] For a user with `language` = `es`, all summaries and all email labels are in Spanish, and there is no vocabulary box.
- [X] For a user with `language` = `en`, the email ends with a vocabulary box of 5 words with Spanish meanings.
- [X] A user with `scope` = `national` receives only articles from feeds with `scope` = `national`.
- [X] A user receives only articles whose feed topic is in their `Topics` rows.
- [X] Changing `send_hour` in `Config` changes the hour of the next scheduled email without editing the workflow.
- [X] With `send_days` = `mon`, a scheduled run on any other day sends nothing.
- [X] With one feed URL replaced by an invalid URL, the run still finishes and the email is sent with the other feeds.
- [X] With the `ollama` container stopped, the email is still sent with original titles and snippets, and without a vocabulary box.
- [X] A user whose topics return zero fresh articles receives no email, and the run does not fail.
- [X] The email renders as a single readable column in Gmail web and in the Gmail mobile app.

## Decisions

- **Yes:** fixed list of RSS feeds in a `Feeds` tab. The user chooses the media, and it needs no API key and has no daily limit.
- **No:** Google News RSS and news APIs. Discarded by the user in favour of chosen sources.
- **Yes:** each feed carries `topic` and `scope`. Matching is exact, so results are predictable and need no AI calls.
- **No:** keyword filtering or AI classification. More empty days or many more calls on slow hardware.
- **Yes:** two data tabs for users (`Users`, `Topics`) instead of five. Scope, count and language are one value per user, so they are columns.
- **Yes:** `scope` accepts `both`. Otherwise a user would need two rows to read national and international news.
- **Yes:** `news_count` is the total per email, spread round-robin across topics. The email size stays constant when topics are added.
- **Yes:** local Ollama with `gemma4:12b` through an HTTP Request node. It is already on the `ia` Docker network, is free, and needs no credential.
- **No:** cloud LLM. Better quality, but costs money and needs a key.
- **Yes:** one Ollama call per article. A failure degrades one article only.
- **No:** one call per user with all articles. Fewer calls, but a single failure would lose the whole email's summaries.
- **Yes:** `think: false` on every Ollama call. `gemma4:12b` thinks by default, and on CPU that took about 125 s per article against about 25 s without it, with no visible loss in the summary.
- **Yes:** `level` column. Summaries written at B2 are the reason the app helps with English practice.
- **Yes:** vocabulary box only for `language` = `en`. Spanish meanings are useless to a Spanish-language reader.
- **Yes:** hourly trigger plus `send_hour` and `send_days` in `Config`. The Schedule Trigger cannot read its time from a sheet.
- **No:** per-user schedule. Not selected for this version.
- **No:** no-repeat log. Not selected. The 24-hour age filter is the only protection against repeats.
- **Yes:** degrade and continue. A broken feed is skipped, a failed summary falls back to the original text, and a user with zero articles gets no email.
- **No:** failure alert email. Not selected. Errors are visible in the n8n executions list.
- **Yes:** Gmail node with OAuth. The Google Cloud project is needed for Sheets anyway.
- **No:** SMTP with app password. A second credential type to maintain.
- **Yes:** one folder per project under `projects/`, with shared infrastructure (`docker-compose.yml`, `specs/`, root `README.md`) at the repo root. More n8n projects will follow, and each keeps its workflows, templates and README together.
- **Yes:** workflow as a JSON file imported by CLI. The workflow is versioned as a file in this folder.
- **Yes:** sheet ID and Ollama URL in a `Settings` node. One place to edit after import, and n8n blocks environment access from expressions by default.
- **Yes:** HTML template inside a Code node of the workflow. A separate template file would be a second copy to keep in sync.
- **Yes:** Google credentials and the sheet are created by hand following `projects/news-digest/README.md`. OAuth consent cannot be automated.
- **Yes:** the starter `Users.csv` uses a placeholder email. The real address is typed in the sheet, not stored in the repo.

## Risks

| Risk | Mitigation |
| --- | --- |
| `gemma4:12b` (8 GB) does not fit in the 4 GB GPU and runs mostly on CPU, about a minute per article. | Calls run one after another with a 300 s timeout. The email arrives some minutes after `send_hour`. `ollama_model` in `Config` allows switching to a smaller model. |
| A feed has no article in the last 24 hours (BBC Technology had none newer than 2 days on 2026-10-04). | Free slots go to the user's other topics. `max_article_age_hours` can be raised in `Config`, at the cost of repeats. |
| RSS snippets are short, so the AI summary may add little or invent details. | The prompt instructs the model to use only the given title and snippet. The link to the original is always shown. |
| n8n is down at `send_hour`. | No email that day. Accepted for this version. A manual run sends it. |
| The model returns invalid JSON. | Treated as a failed call: the article falls back to the original text with `aiOk` = `false`. |
| A typo in a tab name, header or topic value breaks matching silently. | CSV templates guarantee the headers. `projects/news-digest/README.md` lists the allowed values. |
| A feed changes or removes its URL. | The feed is skipped by "continue on error". The URL is fixed in the `Feeds` tab without touching the workflow. |

## What is **not** in this spec

- No-repeat log.
- Per-user schedule.
- Failure alert email.
- Topic detection by keywords or AI.
- News APIs or Google News search.
- Full article text.
- Unsubscribe link or user management form.
- Cloud LLM providers.

Each one of those, if it lands, goes in its own spec.
