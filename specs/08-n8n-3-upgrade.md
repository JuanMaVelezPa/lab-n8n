# SPEC 08 — Upgrade n8n to 3.2.0

> **Status:** Implemented
> **Depends on:** SPEC 06
> **Date:** 2026-11-10
> **Objective:** Upgrade the n8n container from 2.41.6 to 3.2.0 by pinning the image tag in `docker-compose.yml`, after a fresh backup, and leave `cloudflared` untouched. Once 3.2.0 is verified, fix the two deprecations it logs at startup: `WEBHOOK_URL` and the internal task runner mode.

## Why this spec exists

n8n runs 2.41.6 today, from the floating tag `docker.n8n.io/n8nio/n8n`.
SPEC 06 left pinning the image out of scope.
This spec needs a pin anyway, because 3.2.0 is not on any channel tag.
On 2026-11-10, Docker Hub has:

| Tag | Version |
| --- | --- |
| `latest`, `stable` | 2.47.3 |
| `next`, `beta` | 2.48.0 |
| `3.2.0` | 3.2.0, marked pre-release on GitHub |

With the floating tag, a pull would land on 2.47.3, not 3.2.0.

3.0.0 is a major release.
Its breaking changes, from `packages/cli/BREAKING-CHANGES.md` at tag `n8n@3.2.0`, were checked against this instance before writing this spec:

| 3.0.0 change | This instance |
| --- | --- |
| `~/.n8n/binaryData` renamed to `~/.n8n/storage`. n8n does not start if both exist. | Only `storage/` exists in the volume. No action. |
| `$evaluateExpression()` removed from the Code node. | No Code node calls it. |
| Execute Sub-workflow "Local File" and "URL" sources removed. | No workflow uses an Execute Sub-workflow node. |
| Chat hub off by default. | Not used. |
| `N8N_PRE_EXECUTE_ERROR_CREATES_EXECUTION` and `N8N_WORKFLOW_TAGS_DISABLED` removed. | Neither is set. |
| Content-Security-Policy enforced on n8n's pages. | No scripts are injected into n8n. |
| n8n no longer published to npm. | n8n runs from the Docker image. |

The workflows use only these nodes: `code`, `filter`, `gmail`, `googleSheets`, `httpRequest`, `if`, `manualTrigger`, `rssFeedRead`, `scheduleTrigger`, `set`, `splitInBatches`, `splitOut` and `stickyNote`.

## Scope

**In:**

- A check of n8n's built-in v3 migration report on 2.41.6, before the upgrade.
- A fresh backup of the `n8n_data` volume with the procedure in `docs/operations.md`.
- The n8n image in `docker-compose.yml` pinned to `docker.n8n.io/n8nio/n8n:3.2.0`.
- Pulling and recreating only the `n8n` service.
- The "Update n8n" section of `docs/operations.md` rewritten for a pinned tag.
- A documented rollback to 2.41.6 from the backup, used only if a failure cannot be fixed on 3.2.0.
- Added during implementation, applied only after step 8 passes:
  - `WEBHOOK_URL` renamed to `N8N_WEBHOOK_URL`.
  - The task runners moved from internal to external mode, with a new `n8n-runners` service on `n8nio/runners:3.2.0` and a shared `N8N_RUNNERS_AUTH_TOKEN`.

**Out of scope (for future specs):**

- Updating, restarting or pinning `cloudflared`.
- Updating Ollama or the `gemma4:12b` model.
- Any change to workflow logic, workflow files in `projects/*/workflows/`, sheet templates or credentials.
- Running the workflows by hand to test the upgrade.
- Mentioning the n8n version in `README.md` or `docs/architecture.md`.
- Moving the n8n version to `.env`.
- Automated or scheduled backups, and automated updates such as Watchtower or Renovate.
- Turning on the chat hub or any other n8n module that 3.x leaves off.
- A later move from 3.2.0 to a newer version or to the `stable` tag.

## Data model

This feature introduces no new runtime data in n8n or in the sheets.
n8n migrates its own SQLite database (`database.sqlite` in the volume) on the first start of 3.2.0.
After that migration, 2.41.6 cannot open the database, so the backup is the only way back.

**Files**

| Path | Change |
| --- | --- |
| `docker-compose.yml` | The n8n `image` gets the tag `:3.2.0`. After step 8: `WEBHOOK_URL` is renamed, the external runner variables are added to `n8n`, and a new `n8n-runners` service is added. |
| `docs/operations.md` | The "Update n8n" section is rewritten for a pinned tag on both images. |
| `.env.example` | New `N8N_RUNNERS_AUTH_TOKEN` placeholder. |
| `README.md` | Added at the user's request: the `n8n-runners` container in the diagram, text and layout, and a Quick start that sets the runner token and starts `n8n n8n-runners`. No version is mentioned. |
| `docs/setup.md` | Added at the user's request: the first start sets the runner token and starts `n8n n8n-runners`, and `WEBHOOK_URL` is renamed to `N8N_WEBHOOK_URL`. |
| `docs/architecture.md` | Added at the user's request: the `n8n-runners` container in the containers table, the text and the diagram, and `WEBHOOK_URL` renamed to `N8N_WEBHOOK_URL`. No version is mentioned. |
| `.env` (not committed) | New `N8N_RUNNERS_AUTH_TOKEN`, a random 64-character hex string. |
| `backups/n8n_data-<date>.tar.gz` (not committed) | New backup taken just before the upgrade. |

**`docker-compose.yml`** (the only changed line)

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:3.2.0   # was docker.n8n.io/n8nio/n8n
```

The `cloudflared` service keeps `cloudflare/cloudflared:latest` and is not pulled or recreated.

**`docker-compose.yml` after step 10** (the deprecation fixes)

```yaml
services:
  n8n:
    environment:
      # ...unchanged variables...
      - N8N_WEBHOOK_URL=https://lab-n8n.jotaeme.dev/   # was WEBHOOK_URL
      - N8N_RUNNERS_MODE=external
      - N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
      - N8N_RUNNERS_AUTH_TOKEN=${N8N_RUNNERS_AUTH_TOKEN}

  n8n-runners:
    image: n8nio/runners:3.2.0
    container_name: n8n-runners
    restart: unless-stopped
    environment:
      - N8N_RUNNERS_TASK_BROKER_URI=http://n8n:5679
      - N8N_RUNNERS_AUTH_TOKEN=${N8N_RUNNERS_AUTH_TOKEN}
    depends_on:
      - n8n
    networks:
      - ia
```

The broker port 5679 is not published to the host. It is reachable only on the `ia` network, and only with the token.

**"Update n8n" in `docs/operations.md`** (target content, not final text)

1. Pick the version and read the n8n breaking changes between the current version and the new one.
2. Back up the volume with the procedure in "Back up and restore n8n".
3. Change the tag of the n8n `image` and of the `n8n-runners` `image` in `docker-compose.yml`, to the same version.
4. Pull and recreate only n8n and its runners:

   ```bash
   docker compose pull n8n n8n-runners
   docker compose up -d n8n n8n-runners
   ```

5. Check: `docker exec n8n n8n --version` prints the new version, both workflows are still published, and the next scheduled runs succeed.
6. Roll back: put the previous tags back in `docker-compose.yml`, restore the backup taken in step 2, and run `docker compose up -d n8n n8n-runners`.

The section also says that `cloudflared` is updated on its own with `docker compose pull cloudflared` and `docker compose up -d cloudflared`.
The current note that `docker compose pull` also updates `cloudflared` is removed.

Conventions:

- The n8n version lives only in the image tags in `docker-compose.yml`. The `n8n` and `n8n-runners` tags are always the same version.
- Every `docker compose` command in this spec names the `n8n` service (and `n8n-runners` once it exists), so `cloudflared` is never pulled or recreated.

## Implementation plan

1. ~~On 2.41.6, the user opens n8n's v3 migration report in the editor and reads it.~~ **Skipped (2026-11-10).** On 2.41.6 the editor does not show the v3 report. It says "update to the latest v2 version and run the Migration report" instead. The user chose to skip the report and rely on the manual breaking-changes check in "Why this spec exists", over running it through the REST API or adding a stop at 2.47.3.
2. Write down the `cloudflared` container start time with `docker inspect -f '{{.State.StartedAt}}' cloudflared`. Check: a timestamp is printed.
3. Back up the volume with the commands in "Back up" in `docs/operations.md`. Check: `ls -lh backups/` shows `n8n_data-<today>.tar.gz`, larger than 10 MB, and `docker compose ps n8n` shows n8n running again.
4. Change the n8n `image` in `docker-compose.yml` to `docker.n8n.io/n8nio/n8n:3.2.0`. Check: `docker compose config --images` lists `docker.n8n.io/n8nio/n8n:3.2.0` and `cloudflare/cloudflared:latest`.
5. Run `docker compose pull n8n` and `docker compose up -d n8n`. Check: `docker exec n8n n8n --version` prints `3.2.0`. `docker logs n8n` shows no startup error, and `http://localhost:5678` shows the login page.
6. Open the workflow list in n8n. Check: `daily-gospel` and `news-digest` are both published. The `Google Sheets account` and `Gmail account` credentials still exist in **Credentials**.
7. Rewrite the "Update n8n" section of `docs/operations.md` as described in the Data model. Check: `git diff --stat` shows only `docker-compose.yml`, `docs/operations.md` and this spec. (`.env.example` joins the list in step 10.)
8. Wait for the next scheduled run of each workflow. Check: both executions show **Success** in **Executions**, and the emails arrive.
   **Done differently (2026-11-10):** the user ran `news-digest` by hand on 3.2.0 instead of waiting. Execution 1449 succeeded through every node, including the Ollama calls and Send Email, and the email arrived with normal content. The user then stopped the wait. `daily-gospel` was not run on 3.2.0 before step 10, so the scheduled runs in step 11 cover it.
   Two earlier manual runs failed in the Google Sheets nodes with "Client network socket disconnected before secure TLS connection was established". This error is not from 3.2.0: 36 of the 38 failed scheduled runs on 2.41.6 in the previous 14 days were the same error.
9. If step 5, 6 or 8 fails, first try to fix the failure on 3.2.0. If it cannot be fixed, roll back: restore `docker-compose.yml` to the 2.41.6 pin (`docker.n8n.io/n8nio/n8n:2.41.6`), restore the backup from step 3 with "Restore" in `docs/operations.md`, and run `docker compose up -d n8n`. Record the rollback and its cause in this spec. Then `docker exec n8n n8n --version` prints `2.41.6`, and the next scheduled runs succeed.
   **Not needed (2026-11-10):** no failure in steps 5, 6 or 8 called for a rollback.
10. Only after step 8 passes, fix the deprecations as described in the Data model:
    - Generate `N8N_RUNNERS_AUTH_TOKEN` with `openssl rand -hex 32` and add it to `.env`. Add a placeholder to `.env.example`.
    - In `docker-compose.yml`, rename `WEBHOOK_URL` to `N8N_WEBHOOK_URL`, add the three runner variables to `n8n`, and add the `n8n-runners` service.
    - Run `docker compose pull n8n-runners` and `docker compose up -d n8n n8n-runners`.

    Check: `docker logs n8n` shows no `WEBHOOK_URL` or `N8N_RUNNERS_MODE` deprecation and no Python task runner error. `docker logs n8n-runners` shows the launcher connected to the broker. The `cloudflared` start time is unchanged.
11. Wait for the next scheduled run of each workflow. Check: both executions show **Success** and the emails arrive with normal content. If this fails and cannot be fixed, revert step 10 only: restore `docker-compose.yml` to the step 8 state, run `docker compose rm -sf n8n-runners` and `docker compose up -d n8n`. No restore from backup is needed, because step 10 does not migrate the database.
    **Done (2026-11-10):** the scheduled 19:00 UTC runs (1450, 1451) succeeded on the external runner but stopped at Schedule Check, because it was not a send time. The user then ran both workflows by hand to check the full path. News Digest (1452, 6 min) and Daily Gospel (1453, 13 min 45 s) succeeded through Send Email, in the same time as before the upgrade, and both emails arrived with normal content.

## Acceptance criteria

- [x] ~~The v3 migration report on 2.41.6 listed no affected item before the upgrade.~~ Dropped: the report is not available in the 2.41.6 editor (see step 1).
- [x] `backups/` holds an `n8n_data-<date>.tar.gz` taken on the day of the upgrade, before the image changed.
- [x] `docker-compose.yml` has `image: docker.n8n.io/n8nio/n8n:3.2.0` for the n8n service.
- [x] `docker exec n8n n8n --version` prints `3.2.0`.
- [x] `docker inspect -f '{{.State.StartedAt}}' cloudflared` prints the same timestamp as before the upgrade.
- [x] `docker inspect -f '{{.Config.Image}}' cloudflared` prints `cloudflare/cloudflared:latest`.
- [x] `https://lab-n8n.jotaeme.dev` asks for the Cloudflare Access PIN, then shows the n8n login.
- [x] The workflow list shows `daily-gospel` and `news-digest` as published.
- [x] The first scheduled run of each workflow after the upgrade shows **Success** in **Executions** and sends its email. Met with a manual run instead (see step 8): News Digest 1449 sent, and step 11 covered Daily Gospel.
- [x] The "Update n8n" section of `docs/operations.md` says to back up, change the tag, pull and recreate only `n8n`, check, and how to roll back.
- [x] `docs/operations.md` no longer says that `docker compose pull` updates `cloudflared` as part of an n8n update.
- [x] `docker-compose.yml` sets `N8N_WEBHOOK_URL` and `N8N_RUNNERS_MODE=external`, and no longer sets `WEBHOOK_URL`.
- [x] `docker inspect -f '{{.Config.Image}}' n8n-runners` prints `n8nio/runners:3.2.0`, and the container is running.
- [x] `docker logs n8n` after step 10 shows neither deprecation warning.
- [x] The first scheduled run of each workflow after step 10 shows **Success** and sends its email. The scheduled 19:00 UTC runs (1450, 1451) succeeded but were not a send time. Met with manual runs 1452 and 1453, which sent both emails (see step 11).
- [x] `.env.example` has an `N8N_RUNNERS_AUTH_TOKEN` placeholder, and `.env` is not committed.
- [x] `README.md` shows the `n8n-runners` container, and its Quick start starts `n8n n8n-runners` with a random runner token.
- [x] `docs/setup.md` and `docs/architecture.md` show `n8n-runners` and name `N8N_WEBHOOK_URL`, not `WEBHOOK_URL`.
- [x] `git diff main --stat` lists only `docker-compose.yml`, `docs/operations.md`, `docs/setup.md`, `docs/architecture.md`, `.env.example`, `README.md` and `specs/08-n8n-3-upgrade.md`.

## Decisions

- **Yes:** target 3.2.0. Chosen by the user, over `stable` 2.47.3 and `beta` 2.48.0.
- **Yes:** pin the exact tag `:3.2.0`. 3.2.0 is on no channel tag, so a floating tag pulls 2.47.3. The pin also stops a later pull from changing the version by surprise.
- **No:** the `stable` or `latest` tag. Both point to 2.47.3 today.
- **No:** the version in `.env`. One more variable for a value that changes only through a spec.
- **Yes:** leave `cloudflared` untouched. Chosen by the user. The tunnel keeps running, and any failure after the upgrade can only come from n8n.
- **No:** update `cloudflared` in the same run. That would mix an unrelated change into a major upgrade.
- ~~**Yes:** check n8n's own migration report before the upgrade.~~ **Changed during implementation:** the 2.41.6 editor asks to update to the latest v2 first, so the report is skipped. Chosen by the user, over the REST endpoint `/rest/breaking-changes/report?version=v3` and over a stop at 2.47.3. The manual check against the breaking-changes list stands alone. The 3.x rules the report would have checked include `ssrf-default-blocked-ranges` and `task-runner-task-timeout`, so step 8 watches the `httpRequest` calls to Ollama and the Code nodes closely.
- **Yes:** a fresh backup just before the upgrade. The newest backup is from 2026-10-27, and the database migration cannot be undone without one.
- **Yes:** verify with the scheduled runs only. Chosen by the user. Both workflows trigger every hour, so the result shows within an hour, with no extra emails.
- **No:** run the workflows by hand after the upgrade. Not selected.
- **Yes:** fix forward first, and roll back only if the fix fails. Chosen by the user. Emails missed while fixing are accepted.
- **No:** roll back on the first failure. Not selected.
- **Yes:** rewrite only the "Update n8n" section of `docs/operations.md`. Chosen by the user. That is the only doc that changes with a pinned tag.
- **No:** add the version to `README.md` or `docs/architecture.md`. More places to keep in sync on every update.
- **Yes (added during implementation):** fix the two deprecations 3.2.0 logs at startup, in this spec. Chosen by the user, over a separate SPEC 09.
- **Yes:** apply them only after step 8 passes. Chosen by the user. A failure then points to one change, either 3.2.0 or the runner switch.
- **No:** apply them before the first verified run of 3.2.0. Not selected.
- **Yes:** external runners on `n8nio/runners`, pinned to the same tag as n8n. Internal mode is deprecated in 3.x. The runners image also ships Python, which clears the Python runner error.
- **Yes:** go straight from 2.41.6 to 3.2.0, without stopping at 2.47.3. n8n lists no breaking changes between 2.0.0 and 3.0.0, and the 3.0.0 items do not apply to this instance.

## Risks

| Risk | Mitigation |
| --- | --- |
| 3.2.0 is a pre-release and has a bug that breaks a node the workflows use. | The scheduled runs show it within an hour. Fix forward first. If that fails, roll back with step 9. |
| The database migration fails on the first start. | n8n does not start, and `docker logs n8n` shows the error. Roll back with step 9 from the step 3 backup. |
| The backup is corrupt or incomplete. | n8n is stopped during the backup. Step 3 checks the file exists and is larger than 10 MB, and the 2026-10-27 backup is the fallback. |
| 3.x changes how a Code node runs, and a digest or reflection comes out empty or wrong. | Check the content of the emails from step 8, not only the **Success** status. |
| `docker compose up -d n8n` recreates `cloudflared` because it `depends_on` n8n. | The command names only `n8n`, and Compose does not recreate a dependant. Acceptance checks the `cloudflared` start time. |
| The workflows come back unpublished. | Recreating the container does not import, so they stay published. Step 6 checks it and they are published by hand if needed. |
| Emails are missed during the upgrade or while fixing. | Accepted by the user. The upgrade takes minutes, and the workflows run every hour. |

## What is **not** in this spec

- Any change to `cloudflared`.
- Ollama or the model.
- Workflow, sheet or credential changes.
- Manual test runs of the workflows.
- The version in `README.md`, `docs/architecture.md` or `.env`.
- Automated backups or updates.
- Chat hub or other n8n modules.
- Any version after 3.2.0.

Each one of those, if it lands, goes in its own spec.
