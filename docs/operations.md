# Operations

Day-to-day operation of the stack: moving workflows between the repo and n8n, backups, updates and troubleshooting.
It assumes the [shared setup](setup.md) is done.

Commands are run from the repo root.

## Import a workflow

The workflows in `projects/*/workflows/` are the source of truth.
n8n runs in a container that cannot see this folder, so the file is copied into the container first and imported from there:

```bash
docker cp projects/<project>/workflows/<file>.json n8n:/tmp/<file>.json
docker exec n8n n8n import:workflow --input=/tmp/<file>.json
```

Each project README has the exact commands, with its file name.

Importing a file with an ID that already exists replaces that workflow in n8n.
Import also changes the live workflow in two ways:

- It **unpublishes** the workflow. The schedule stops until you click **Publish** again.
- It **drops `sheetId` and the selected credentials**. The repo files hold the placeholder `<Google Sheet document ID>` and refer to the credentials by name only, so every import must be finished by hand.

After every import, open the workflow in n8n and:

1. Paste the document ID of your sheet into `sheetId` in the `Settings` node.
2. Select the `Google Sheets account` and `Gmail account` credentials in the nodes listed in the project README.
3. Save.
4. Click **Publish**.

Check: the workflow list shows the workflow as published.

## Export a workflow to the repo

After a change in the editor, export the workflow and bring it back into the repo.
The IDs are fixed in the repo files: `newsDigest0000001` and `dailyGospel000001`.
`docker exec n8n n8n list:workflow` lists them.

```bash
docker exec n8n n8n export:workflow --id=<workflow ID> --pretty --output=/tmp/export.json
docker cp n8n:/tmp/export.json backups/export.json
```

The export holds your real sheet ID, the credential IDs and fields that only make sense in your n8n.
Clean it before it replaces the repo file:

```bash
jq '.[0]
  | {id, name, active: false, nodes, connections, settings, pinData}
  | .nodes |= map(
      del(.webhookId)
      | (if .credentials then .credentials |= map_values({name}) else . end)
      | (if .name == "Settings" then .parameters.assignments.assignments |= map(if .name == "sheetId" then .value = "<Google Sheet document ID>" else . end) else . end))' \
  backups/export.json > projects/<project>/workflows/<file>.json
```

This keeps only the fields of the repo format, sets `sheetId` back to the placeholder and keeps only the credential names.
It needs `jq` on the host.

Check: `git diff` shows only the change you made in the editor, and `git grep -n sheetId -- projects` shows only the placeholder.

## Back up and restore n8n

All the n8n state lives in the `n8n_data` volume: workflows, executions and the encrypted credentials.
Back it up into `backups/`, which is in `.gitignore`.

The commands below reach the volume through the `n8n` container, so they work whatever name Compose gave the volume.
Compose names it `<project>_n8n_data`, where `<project>` is the folder name unless `COMPOSE_PROJECT_NAME` is set. `docker volume ls` shows it.

### Back up

```bash
mkdir -p backups
docker compose stop n8n
docker run --rm --volumes-from n8n -v "$PWD/backups:/backup" alpine \
  tar czf /backup/n8n_data-$(date +%F).tar.gz -C /home/node/.n8n .
docker compose start n8n
```

Stopping n8n first makes sure the database is not written during the copy.

Check: `ls -lh backups/` shows the new `.tar.gz` file.

The credentials in the backup are encrypted with the key stored in the same volume, so the backup restores with working credentials.
Keep it private all the same.

### Restore

This replaces everything in the volume with the backup.

```bash
docker compose stop n8n
docker run --rm --volumes-from n8n -v "$PWD/backups:/backup" alpine \
  sh -c 'find /home/node/.n8n -mindepth 1 -delete && tar xzf /backup/n8n_data-<date>.tar.gz -C /home/node/.n8n'
docker compose start n8n
```

Check: open http://localhost:5678, sign in, and see the workflows and credentials of the backup.

## Update n8n

```bash
docker compose pull
docker compose up -d
```

Back up the volume first. A new n8n version can migrate the database, and an older image cannot open it afterwards.

Check: `docker exec n8n n8n --version` prints the new version, and the workflows are still published.

## Ollama

Both projects call Ollama with `POST /api/generate` on `http://ollama:11434`.
The model is the value of `ollama_model` in the `Config` tab of each sheet. The templates use `gemma4:12b`.

- **Model choice.** A new model must be pulled first with `docker exec ollama ollama pull <model>`. A smaller model is faster but writes worse text, especially in Spanish and for the vocabulary box.
- **CPU and GPU.** On CPU the model works, but each call takes from seconds to minutes. Each project README gives the times on CPU. A GPU mostly speeds up reading the prompt. A GPU with less memory than the model holds only part of it, so writing the answer stays close to CPU speed.
- **Timing.** The calls run one after another, so the email arrives some minutes after `send_hour`. Give each project a different `send_hour` so they do not compete for Ollama.
- **Time zones.** n8n runs in `America/Bogota` (`TZ` and `GENERIC_TIMEZONE` in `docker-compose.yml`), and `send_hour` uses that time. `docker logs ollama` prints times in UTC, five hours ahead. A run at 05:00 in Bogota shows at 10:00 in the Ollama logs.

Check which processor a model uses while a workflow runs:

```bash
docker exec ollama ollama ps
```

## When an email does not arrive

1. Open **Executions** in n8n and find the run of that hour.
   - No run at all: the workflow is not published. Click **Publish**.
   - A run that stopped early: the hour or the day is not in `send_hour` and `send_days` of the `Config` tab.
   - A run with an error: open it and look at the node in red.
2. Open the `Build Plan` node of the run. It shows who was going to get an email, and why a row was skipped.
3. Read the "When something fails" section of the project:
   - [News digest](../projects/news-digest/README.md#when-something-fails)
   - [Daily gospel](../projects/daily-gospel/README.md#when-something-fails)

A credential that shows an error in **Credentials** must be reconnected with **Sign in with Google**. See [step 6 of the setup](setup.md#6-create-the-credentials-in-n8n).
