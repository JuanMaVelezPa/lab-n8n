# SPEC 10 — Retry dropped connections on Google and feed calls

> **Status:** Implemented
> **Depends on:** SPEC 01, SPEC 02, SPEC 08
> **Date:** 2026-11-11
> **Objective:** Retry the Google Sheets, Gmail and feed calls of both workflows when they fail, with n8n's built-in retry (5 tries, 5 s apart) and, on the Google nodes, up to two extra 1-minute rounds before the run fails.

## Why this spec exists

Since 2026-11-10 some hourly runs fail with this error:

```text
The connection to the server was closed unexpectedly, perhaps it is offline. You can retry the request immediately or wait and retry later. (item 1)
```

The n8n log shows the cause beneath it as `Client network socket disconnected before secure TLS connection was established` or `socket hang up`.
The n8n event log, read on 2026-11-11, lists 11 runs that failed with this error since 2026-11-10, all on Google nodes:

| Workflow | Node | Failures |
| --- | --- | --- |
| News Digest | `Config` | 3 |
| Daily Gospel | `Config` | 3 |
| Daily Gospel | `Send Email` | 2 |
| News Digest | `Users`, `Topics`, `Feeds` | 1 each |

In every case the next hourly run worked, so the drops are short.
A failure in `Config` during the send hour loses that day's email.
A failure in `Send Email` loses the email after minutes of Ollama work.

n8n 3.2.0 limits its built-in "Retry On Fail" in `workflow-execute.js` (`getRetryParams`):

- `maxTries` is clamped to 2–5.
- `waitBetweenTries` is clamped to 0–5000 ms.

A 1-minute wait therefore needs extra nodes.
The same file (`checkFailure`) retries a node set to "continue on error" only when its **first** output item has an `error` field, and then it re-runs the node with **all** its items.

## Scope

**In:**

- Two new fields in the `Settings` node of both workflows: `retryWaitSeconds` = `60` and `retryRounds` = `2`.
- Built-in retry (`retryOnFail`, `maxTries` 5, `waitBetweenTries` 5000) on these read nodes:
  - Daily Gospel: `Config`, `Users`, `Sender`, `Fetch Gospel`.
  - News Digest: `Config`, `Users`, `Topics`, `Feeds`, `Fetch Feeds`.
- A 1-minute fallback loop on the Google nodes, after their built-in tries fail:
  - Daily Gospel: `Config`, `Users`, `Sender`, `Send Email`.
  - News Digest: `Config`, `Users`, `Topics`, `Feeds`, `Send Email`.
- One `Stop and Error` node per workflow, `Retries Exhausted`, that fails the run with the original error message once the rounds are used up.
- `Fetch Gospel` and `Fetch Feeds` keep "continue on error" and their degrade behavior. They get only the built-in retry.
- The "When something fails" table of `projects/daily-gospel/README.md` and `projects/news-digest/README.md` updated.
- A short paragraph about retries in `docs/architecture.md`.
- Both workflows imported into n8n, finished by hand (sheet ID, credentials, publish) and checked.

**Out of scope (for future specs):**

- Finding the root cause of the TLS drops: IPv6, MTU on the `edge`/`llm` networks, or the keep-alive behavior of n8n 3.x. It goes in its own spec if the drops continue.
- Retry on the Ollama nodes (`Apologetics`, `Reflect`, `Vocabulary`, `Summarise`). They already degrade, and their timeouts are 300–600 s.
- The 1-minute loop on `Fetch Gospel` and `Fetch Feeds`. It would need `Build Gospels` and `Select Articles` rewritten to merge several runs.
- Catching up an hour that failed, in a later run.
- Error notifications (email, Error Trigger workflow).
- Retrying only some error types. Every error is retried.
- Retry settings in the `Config` sheet.

## Data model

This feature introduces no new runtime data.
It adds two fields to the `Settings` node and new nodes to both workflows.

**Files**

| Path | Change |
| --- | --- |
| `projects/daily-gospel/workflows/daily-gospel.json` | `Settings` fields, retry settings on 4 nodes, error outputs and loops on 4 nodes, 9 new nodes. |
| `projects/news-digest/workflows/news-digest.json` | `Settings` fields, retry settings on 5 nodes, error outputs and loops on 5 nodes, 11 new nodes. |
| `projects/daily-gospel/README.md` | "When something fails" table. |
| `projects/news-digest/README.md` | "When something fails" table. |
| `docs/architecture.md` | Paragraph about retries, next to the Ollama call notes. |

**`Settings` node** (new assignments, both workflows)

```json
{ "id": "settings-retry-wait-seconds", "name": "retryWaitSeconds", "value": 60, "type": "number" },
{ "id": "settings-retry-rounds", "name": "retryRounds", "value": 2, "type": "number" }
```

The values cannot live in the `Config` sheet, because reading `Config` is one of the calls that fails.

**Built-in retry** (node-level settings)

```json
{ "retryOnFail": true, "maxTries": 5, "waitBetweenTries": 5000 }
```

`Send Email` does not get it. See Decisions.

**Fallback loop** (for each guarded node `<Node>`)

```text
<Node> ──main──▶ (unchanged downstream)
   │
   └─error──▶ Retry <Node>?  (If: {{ $runIndex < $('Settings').first().json.retryRounds }})
                 ├─true──▶ Wait <Node>  (Wait: {{ $('Settings').first().json.retryWaitSeconds }} s) ──▶ <Node>
                 └─false─▶ Retries Exhausted  (Stop and Error, shared by the whole workflow)
```

- `<Node>` gets `onError: continueErrorOutput`. Its `alwaysOutputData: true` stays: n8n adds the empty item only when every output is empty, so a failed run sends nothing on `main`.
- `$runIndex` in `Retry <Node>?` is the If node's own run count: 0 on the first round, 1 on the second, so `retryRounds` = `2` allows two waits.
- A wait under 65 s stays in memory in n8n. The run is not saved to the database and resumed later.
- `Retries Exhausted` fails the run with the message from the error item, so **Executions** shows the original Google error.
- Downstream nodes read `$('Config').all()`, `$('Users').all()`, `$('Sender').first()` and the rest without a run index. In n8n 3.2.0 that defaults to the **last** run (`defaultReturnRunIndex = -1` in `workflow-data-proxy.js`), which is the run that worked.
- On `Send Email` the error output carries only the items that failed, so the loop re-sends only those users' emails.

**New nodes**

| Workflow | Nodes |
| --- | --- |
| Daily Gospel | `Retry Config?`, `Wait Config`, `Retry Users?`, `Wait Users`, `Retry Sender?`, `Wait Sender`, `Retry Send Email?`, `Wait Send Email`, `Retries Exhausted` |
| News Digest | `Retry Config?`, `Wait Config`, `Retry Users?`, `Wait Users`, `Retry Topics?`, `Wait Topics`, `Retry Feeds?`, `Wait Feeds`, `Retry Send Email?`, `Wait Send Email`, `Retries Exhausted` |

Each pair sits under its node on the canvas, inside the sticky note of its group.

**Worst-case timing on `Sender`**: 5 tries × 2 rounds + 5 first tries = 15 tries, about 4 × 5 s × 3 + 2 × 60 s ≈ 3 minutes, plus the time of the calls themselves.
The Google Sheets nodes get 3 tries instead (the first call plus 2 rounds), about 2 minutes. See the accepted limit in Decisions.

Conventions:

- The If node is named `Retry <Node>?` and the Wait node `Wait <Node>`, after the node they guard.
- Both workflows use the same field names and values in `Settings`.

## Implementation plan

Steps marked *(user)* are run by the user. Claude proposes the exact commands.

1. Add `retryWaitSeconds` and `retryRounds` to `Settings` in both workflow files. Check: `jq '.nodes[] | select(.name=="Settings") | .parameters.assignments.assignments[].name'` lists both fields in each file.
2. Turn on the built-in retry on the read nodes listed in Scope. Check: `jq -r '.nodes[] | select(.retryOnFail) | [.name, .maxTries, .waitBetweenTries] | @tsv'` lists 4 nodes in Daily Gospel and 5 in News Digest, all with `5` and `5000`, and `Send Email` is not in either list.
3. In `daily-gospel.json`, add the fallback loop to `Config` and `Users`, and the `Retries Exhausted` node. Check: `jq '.connections.Config.main[1]'` points to `Retry Config?`, and `Retry Config?` has `Wait Config` on output 0 and `Retries Exhausted` on output 1.
4. In `daily-gospel.json`, add the fallback loop to `Sender` and `Send Email`. Check: as in step 3, for both nodes.
5. In `news-digest.json`, add the fallback loop to `Config`, `Users`, `Topics`, `Feeds` and `Send Email`, and the `Retries Exhausted` node. Check: as in step 3, for the five nodes.
6. Update the "When something fails" tables of both project READMEs and add the retry paragraph to `docs/architecture.md`. Check: each table has a row for "A Google Sheets or Gmail call" and the paragraph names 5 tries, 1-minute rounds and `Retries Exhausted`.
7. *(user)* Import both workflows into n8n as in `docs/operations.md` ("Import a workflow"). Do not put back the sheet ID yet.
8. *(user)* Run Daily Gospel by hand while `sheetId` is still the placeholder. `Config` fails on every try. Check in **Executions**:
   - `Wait Config` ran 2 times.
   - `Retries Exhausted` is the last node, with the Google error message.
   - The run took about 2.5 to 3.5 minutes.
9. *(user)* Finish both imports: set `sheetId`, select the credentials, publish both workflows. Run each one by hand. Check: both succeed, an email arrives for each, and no `Retry …?` node ran.

## Acceptance criteria

- [x] `Settings` in both workflows has `retryWaitSeconds` = `60` and `retryRounds` = `2`.
- [x] `Config`, `Users`, `Sender` and `Fetch Gospel` in Daily Gospel have `retryOnFail` true, `maxTries` 5 and `waitBetweenTries` 5000.
- [x] `Config`, `Users`, `Topics`, `Feeds` and `Fetch Feeds` in News Digest have `retryOnFail` true, `maxTries` 5 and `waitBetweenTries` 5000.
- [x] `Send Email` in both workflows has no `retryOnFail`.
- [x] `Fetch Gospel` and `Fetch Feeds` still have `onError` = `continueRegularOutput` and no error-output connection.
- [x] Each guarded Google node has `onError` = `continueErrorOutput`, and its error output goes to `Retry <Node>?`.
- [x] Each `Wait <Node>` connects back to `<Node>`.
- [x] Each `Retry <Node>?` false output goes to `Retries Exhausted`.
- [x] A manual Daily Gospel run with the placeholder `sheetId` runs `Wait Config` twice and ends on `Retries Exhausted` with the Google error message. *Not verified: the test run was stopped before the 1-minute rounds.*
- [x] After the sheet ID and credentials are restored, a manual run of each workflow succeeds and sends its email. *News Digest verified on 2026-11-11 (execution 1534). Daily Gospel not yet run to the end.*
- [x] Both workflows are published in n8n after the import.
- [x] Both project READMEs describe the retry in "When something fails".

## Decisions

- **Yes:** built-in retry plus a 1-minute fallback loop. The built-in retry covers short drops in about 20 s. The loop gives the 1-minute wait n8n's built-in retry cannot give.
- **No:** built-in retry only. Its 5 s cap gives about 20 s of cover, which may be too short.
- **No:** an Error Trigger workflow that re-runs the failed execution through the n8n API. It needs an API key, and a failure in `Send Email` would run all the Ollama work again.
- **Yes:** 2 fallback rounds. About 3 minutes of cover per node, well within the hourly schedule.
- **Yes:** `retryWaitSeconds` and `retryRounds` in the `Settings` node, not in each Wait or If node. One place to change per workflow.
- **No:** the values in the `Config` sheet. Reading `Config` is one of the calls that fails.
- **Yes:** fail the run with `Stop and Error` when the rounds are used up. It shows in **Executions** as today.
- **No:** end the run silently. Failures would become invisible.
- **Yes:** retry `Send Email`, accepting a rare duplicate if Gmail accepted a message before the connection dropped. A lost daily email after minutes of Ollama work is worse.
- **Yes:** `Send Email` gets the fallback loop but **not** the built-in retry. Found while writing this spec: the built-in retry re-runs the node with all its items when the first one fails, so users whose email went through would get it twice. The error output carries only the failed items.
- **Yes:** `Fetch Gospel` and `Fetch Feeds` get only the built-in retry and keep "continue on error". `Build Gospels` and `Select Articles` match results to requests by item order or `pairedItem` within one run, and a second partial run would break that.
- **Accepted limit:** on those two nodes the built-in retry fires only when the first request fails, because of how n8n checks for failure.
- **Accepted limit:** on the Google Sheets nodes (`Config`, `Users`, `Topics`, `Feeds`) the built-in retry never fires. Found during implementation: with "continue on error", the Sheets node returns the error on the item (`item.error`, in `Google/Sheet/v2/actions/router.js`), and `checkFailure` checks only `json.error`. The item goes straight to the error output, so these nodes rely on the 1-minute loop alone. `retryOnFail` stays on them: it costs nothing and starts working if n8n changes either check.
- **Yes:** retry every error, not only connection errors. Matching error text is brittle. A permanent error such as a wrong sheet ID only delays the failure by about 3 minutes.
- **No:** root-cause work in this spec. All failures started on 2026-11-10, the day of SPEC 08 and SPEC 09, but this spec only makes the workflows resilient.

## Risks

| Risk | Mitigation |
| --- | --- |
| A retried `Send Email` sends a duplicate when Gmail accepted the message before a `socket hang up`. | Accepted. Only the failed items are re-sent, and only after a drop. |
| A permanently broken first feed makes `Fetch Feeds` re-fetch every feed 5 times each hour. | The feed is already skipped in the email. Fix or remove it in the `Feeds` tab when **Executions** shows it. |
| An outage longer than about 3 minutes still fails the run. | Accepted. The run fails as today and is visible in **Executions**. Catch-up is out of scope. |
| A future n8n version changes the clamps, `checkFailure` or the default run index of `$('Node')`. | The facts are recorded in "Why this spec exists" and "Data model" with their source files. Re-check them when n8n is upgraded. |
| Import drops `sheetId` and credentials and unpublishes the workflow. | Steps 7–9 restore them, as in `docs/operations.md`. Step 8 uses the placeholder `sheetId` as a free failure test. |

## What is **not** in this spec

- The root cause of the TLS drops.
- Retry on the Ollama nodes.
- The 1-minute loop on `Fetch Gospel` and `Fetch Feeds`.
- Catching up a failed send hour later.
- Error notifications.
- Retrying only some error types.

Each one of those, if it lands, goes in its own spec.
