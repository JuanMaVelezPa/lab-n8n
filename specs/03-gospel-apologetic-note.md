# SPEC 03 — Apologetic note in the daily gospel email

> **Status:** Implemented
> **Depends on:** SPEC 02
> **Date:** 2026-10-13
> **Objective:** Add a step to the Daily Gospel workflow where local Ollama decides, once per language, whether the day's passage is hard to understand and, only when it is, writes a two-line apologetic note (context and meaning) that is shown after the gospel text and given to the reflection prompt.

## Why this spec exists

Some gospels have a custom, an image or a saying that a modern reader misreads or finds shocking.
The reflection of SPEC 02 applies the passage to life, but it does not stop to explain what the passage means.
This spec adds that explanation from a Catholic apologetic point of view, in two easy lines.
Many gospels are clear on their own, so the note appears only on the days that need it.

This spec modifies the workflow of SPEC 02.
It adds no new project, sheet or tab.

## Scope

**In:**

- A new step in `projects/daily-gospel/workflows/daily-gospel.json`, placed between `Build Gospels` and `Reflection Prompts`.
- One Ollama call per language needed that day (`en`, `es`), so at most 2 extra calls per run.
- The model decides per gospel whether the note is needed, and returns that decision as a `needed` flag.
- The note has exactly two lines: `context` (historical or cultural background needed to read the passage) and `meaning` (what the passage means in Catholic teaching).
- The English note is written once in simple English and shared by all English levels.
- The note has no citations: no Catechism numbers, no authors, no quotes and no verses outside the given text.
- A new section `Apologetic note` (`Nota apologética` in Spanish) in the email, after the gospel text and its attribution, before the question-and-answer pairs.
- The reflection prompt receives the two lines of the note when the note is valid and needed.
- A new key `apologetics` in the `Config` tab, with `on` or `off`, to disable the whole step.
- The new row in `projects/daily-gospel/sheets/Config.csv`.
- The footer text changed to say that the note and the reflection were written by an AI.
- Degrade and continue: a failed call or a `needed` = `false` answer leaves the section out, and the rest of the email is unchanged.
- `projects/daily-gospel/README.md` updated with the new key and the new section.

**Out of scope (for future specs):**

- A note per English level or per audience.
- A per-row column in `Users` to opt in or out.
- Citations of the Catechism, Church Fathers or other bible verses.
- A notice in the email on days without a note.
- Using the commentary that Evangelizo publishes as a source.
- Answers to specific objections from other religions or from atheism.
- Vocabulary words taken from the note.
- A longer study section or links to further reading.

## Data model

**Tab `Config`** — one new key/value pair. The rest of the tab is as in SPEC 02.

```csv
key,value
apologetics,on
```

- `apologetics`: `on` or `off`. Upper and lower case are treated as equal.
- With `off`, no apologetics call is made and no email shows the section.
- A key that is missing, empty or has any other value counts as `on`.

**Ollama response** — one call per language, requested with `stream: false`, `think: false` and a JSON schema of the expected answer in `format`, on `POST /api/generate`, as in SPEC 02.

```json
{
  "needed": true,
  "context": "one sentence",
  "meaning": "one sentence"
}
```

**Note object** — one per language, after validation:

```json
{
  "language": "en",
  "needed": true,
  "context": "In Jesus' time, washing before a meal was a religious rite, not a matter of hygiene.",
  "meaning": "Jesus teaches that God looks at the heart first, and that outer practices are worth nothing without love.",
  "ok": true
}
```

Conventions:

- The call is made only for a language that is needed by an active row and whose Gospel object has `ok` = `true`.
- The model comes from `ollama_model` in `Config`. The timeout is 600 s, with "continue on error".
- The prompt receives only the cleaned gospel text, the reference and the language.
- The prompt asks for `needed` = `true` only when the passage has a custom, image, expression or saying that a modern reader would likely misread or find shocking.
- The prompt asks for one sentence per line, in plain words, at B1 level for English.
- The prompt forbids citations, author names, paragraph numbers and verses that are not in the given text.
- A response is valid when `needed` is a boolean and, if `needed` is `true`, `context` and `meaning` are non-empty strings of at most 300 characters each.
- `ok` is `false` when the call fails or the response is not valid.
- The section is shown in a language block only when the Note object of that language has `ok` = `true` and `needed` = `true`.
- When `needed` is `false`, `context` and `meaning` are ignored.
- The reflection prompt of a variant receives `context` and `meaning` of its language under the same condition. Otherwise the prompt is the one of SPEC 02.
- The reflection of a variant is generated whether or not the note exists.
- The note is shown even when the reflection of the row's variant failed.
- English and Spanish notes are generated independently, each from its own gospel text.
- Section heading: `Apologetic note` in the English block, `Nota apologética` in the Spanish block.
- The section has no sub-labels. `context` is the first line and `meaning` is the second.
- The vocabulary call is unchanged. Its words still come from the English gospel text and the English reflection.

## Implementation plan

1. Add the row `apologetics,on` to `projects/daily-gospel/sheets/Config.csv` and to the `Config` tab of the `Daily Gospel` sheet. Check: a manual run of the current workflow still sends the same email as before.
2. Add a Code node `Apologetics Prompts` after `Build Gospels` that reads `apologetics` from `Config` and builds one prompt per needed language with `ok` gospel. Check: the starter row (`both`) produces two prompts, and `apologetics` = `off` produces none.
3. Add the loop `Loop Over Notes`, the If node `Wants Note?` and the HTTP Request node `Apologetics` (timeout 600 s, continue on error, JSON schema in `format`). Check: a manual run returns a JSON with `needed`, `context` and `meaning` for each language.
4. Add a Code node `Apply Apologetics` that validates each response and builds the Note objects. Connect its output to `Reflection Prompts`, so the run continues with or without notes. Check: the output has one Note object per language with `ok` set, and the email is still sent unchanged.
5. Change `Reflection Prompts` to add `context` and `meaning` to the prompt of each variant whose language has a valid and needed note. Check: the prompt text of `en|B2|men` contains the two lines on a day with a note, and is equal to the SPEC 02 prompt on a day without one.
6. Change `Render Email` to show the section after the attribution line of each language block, under the conditions of the Data model, and change the footer text. Check: copying the output HTML into a browser shows the heading and two lines in the right block.
7. Add the sticky note `Note Apologetics` in the workflow. Update `projects/daily-gospel/README.md` with the `apologetics` key and the new section, and the description of the project in the root `README.md`. Re-export the final workflow to `projects/daily-gospel/workflows/daily-gospel.json`.

## Acceptance criteria

- [x] `projects/daily-gospel/sheets/Config.csv` contains the row `apologetics,on`.
- [x] `docker exec n8n n8n import:workflow --input=<file>` imports `projects/daily-gospel/workflows/daily-gospel.json` without errors.
- [x] A manual run with the starter row (`both`) shows exactly two executions of the `Apologetics` node, one for `en` and one for `es`.
- [x] A manual run with active rows `en|B1|general`, `en|B2|men` and `en|C1|women` shows exactly one execution of the `Apologetics` node.
- [x] With a Note object where `ok` = `true` and `needed` = `true`, the email shows the heading `Apologetic note` with exactly two lines, after the attribution line and before the first question.
- [x] In the Spanish block, the heading is `Nota apologética` and both lines are in Spanish.
- [x] With a Note object where `needed` = `false`, the email has no `Apologetic note` heading and no empty box in its place.
- [x] Two English rows with different `level` values receive the same note text on the same day.
- [x] The note contains no Catechism number, no author name and no bible reference other than the day's own.
- [x] On a day with a valid and needed note, the prompt sent by the `Reflect` node contains the `context` and `meaning` text of that language.
- [x] On a day where `needed` = `false`, the prompt sent by the `Reflect` node contains no note text.
- [x] With `apologetics` = `off` in `Config`, the `Apologetics` node is not executed, and the email has no note section.
- [x] With the `apologetics` row deleted from `Config`, the step runs as with `on`.
- [x] With the `Apologetics` node URL replaced by an invalid URL, the email is still sent with the gospel, the reflection and the vocabulary, and without the note section.
- [x] With the `ollama` container stopped, the email is still sent with the liturgical day, reference, gospel text and attribution, as in SPEC 02.
- [x] A response with an empty `context` and `needed` = `true` gives `ok` = `false`, and the section is not shown.
- [x] The footer states that the note and the reflection were written by an AI.
- [x] `projects/daily-gospel/README.md` documents the `apologetics` key and its two values.

## Decisions

- **Yes:** the AI decides per gospel whether the note is needed. Chosen by the user: many passages are clear and need no note, and nobody has to review the gospel each day.
- **No:** the note on every gospel. Clear passages would get a redundant note.
- **Yes:** `needed` only when the passage is hard. Chosen by the user over "usually yes".
- **Yes:** two lines, context first and meaning second. Chosen by the user over "difficulty + meaning" and "meaning only".
- **Yes:** a separate Ollama call, one per language. The note depends on the passage and not on level or audience, so 2 calls cover every row.
- **No:** extra fields in the reflection call. The note would be generated again in each variant, up to 12 times, and two variants could disagree on `needed` on the same day.
- **No:** changing the reflection schema. That schema already needed a fix in SPEC 02, and it stays as it is.
- **Yes:** the call runs before the reflection and its two lines go into the reflection prompt. The dialogue then builds on the same meaning and does not contradict the note.
- **No:** a display-only note. Cheaper by nothing, and the reflection could read the passage in another way.
- **Yes:** one simple English version for all levels. The user asked for two easy lines, and one call serves `B1`, `B2` and `C1`.
- **No:** one note per level. Up to 3 English calls for two sentences.
- **Yes:** no citations of any kind. `gemma4:12b` tends to invent reference numbers, and SPEC 02 already forbids verses outside the given text.
- **No:** named sources without numbers, or full citations. Not selected.
- **Yes:** the section goes after the gospel text. The reader gets the context while the passage is fresh, before the reflection.
- **No:** before the gospel text or at the end of the email. Not selected.
- **Yes:** nothing is shown when the note is not needed or the call fails. The email looks as it does today.
- **No:** a notice such as "today's passage needs no extra context". Not selected.
- **Yes:** `apologetics` key in `Config` with `on` and `off`. Chosen by the user: with `off` the run is as fast as before this spec.
- **No:** a per-row column in `Users`. More to maintain, and the calls would still run if one row wants the note.
- **Yes:** a missing or unknown value counts as `on`. The sheet that already exists keeps working before the row is added, and the feature is on by default.
- **Yes:** heading `Apologetic note` / `Nota apologética`. Chosen by the user over "Understanding the passage" and "Context and meaning".
- **Yes:** each language decides `needed` on its own, from its own gospel text. It follows the SPEC 02 rule that each language is generated independently.
- **No:** one shared decision translated to the other language. It needs a third call or a translation step.
- **Yes:** the note is shown even when the reflection failed. The two calls are independent, and the note explains the gospel text that is always shown.
- **Yes:** a limit of 300 characters per line in the validation. It keeps the note at two lines when the model writes too much.
- **Yes:** the footer names the note together with the reflection. Readers must not take either as church teaching.
- **Yes:** the vocabulary call is not changed. The note is in simple English and adds few hard words.
- **Yes:** a new spec that depends on SPEC 02, which stays as `Implemented`. The history of what was built first is kept.

## Risks

| Risk | Mitigation |
| --- | --- |
| `gemma4:12b` writes a context or a meaning that is historically or doctrinally wrong. | The prompt gives only the day's text and forbids citations. The real gospel text is shown above the note. The footer says the note is AI-written. `apologetics` = `off` removes the step. |
| The model says `needed` = `true` almost every day, or almost never. | The prompt states the rule with the four cases (custom, image, expression, saying). The rule is one text in the `Apologetics Prompts` node and can be tuned after some days of use. |
| The English note says `needed` and the Spanish one does not, so a `both` email shows the section in one block only. | Accepted. Each block is complete on its own. |
| A wrong note is passed to the reflection, and the reflection repeats the mistake. | The note is two sentences beside the full gospel text in the prompt. With `apologetics` = `off` the reflection prompt is the one of SPEC 02. |
| Two more calls make the run longer on CPU. | The answer is two sentences, shorter than a reflection. The calls run one after another with the 600 s timeout. The email arrives some minutes later after `send_hour`. |
| The model returns invalid JSON or lines that are too long. | The JSON schema in `format` forces the shape. A response that fails validation counts as a failed call and the section is left out. |
| The `apologetics` row is not added to the existing sheet. | A missing key counts as `on`, so the step still runs. |

## What is **not** in this spec

- Note per English level or per audience.
- Per-row opt-in or opt-out.
- Citations of the Catechism, Church Fathers or other verses.
- Notice on days without a note.
- Evangelizo commentary as a source.
- Answers to objections from other religions or from atheism.
- Vocabulary from the note.
- Longer study section or further reading.

Each one of those, if it lands, goes in its own spec.
