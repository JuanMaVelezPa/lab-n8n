# Operations

Day-to-day operation of the stack: moving workflows between the repo and n8n, backups, updates, the Cloudflare Tunnel and troubleshooting.
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

All the n8n state lives in the `n8n_n8n_data` volume: workflows, executions and the encrypted credentials.
Back it up into `backups/`, which is in `.gitignore`.

The volume name is fixed to `n8n_n8n_data` in `docker-compose.yml` (`name:` under `volumes`), so it does not follow the Compose project `lab-n8n`.
It keeps the name it got under the old project `n8n`, and Compose prints a warning on `up` that the volume was created for project `n8n`. The warning is expected and harmless.
Never run `docker compose down -v`: `-v` deletes this volume.

The commands below reach the volume through the `n8n` container, so they need it to exist. `docker volume ls` shows the volume.

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

The n8n version is pinned in `docker-compose.yml`, in the `image` tag of two services: `n8n` and `n8n-runners`.
Both tags are always the same version. A plain `docker compose pull` does not change the version.

1. Pick the new version and read the n8n breaking changes between the current version and the new one.
   For a major version, also run n8n's migration report (**Settings → Migration Report**) on the latest release of the current major.
2. Back up the volume with "Back up" above.
   A new n8n version can migrate the database, and an older image cannot open it afterwards, so this backup is the only way back.
3. Change the tag of both images in `docker-compose.yml` to the new version:

   ```yaml
   n8n:
     image: docker.n8n.io/n8nio/n8n:<version>
   n8n-runners:
     image: n8nio/runners:<version>
   ```

4. Pull and recreate only n8n and its runners:

   ```bash
   docker compose pull n8n n8n-runners
   docker compose up -d n8n n8n-runners
   ```

5. Check:
   - `docker exec n8n n8n --version` prints the new version.
   - `docker logs n8n` shows no startup error.
   - Both workflows are still published.
   - The next scheduled runs show **Success** in **Executions**, and the emails arrive.
6. Roll back, only if a failure cannot be fixed on the new version:
   - Put the previous tags back in `docker-compose.yml`.
   - Restore the backup from step 2 with "Restore" above.
   - Run `docker compose up -d n8n n8n-runners`.

`cloudflared` and Ollama are not in this compose file, so these commands never touch them.

## Cloudflare Tunnel

The tunnel and its `cloudflared` container live in [jotaeme-ingress](https://github.com/JuanMaVelezPa/jotaeme-ingress), which publishes n8n at `https://lab-n8n.jotaeme.dev` behind Cloudflare Access.
Its status, logs, updates and token rotation are in the [jotaeme-ingress operations guide](https://github.com/JuanMaVelezPa/jotaeme-ingress/blob/main/docs/operations.md), including [Rotate the tunnel token](https://github.com/JuanMaVelezPa/jotaeme-ingress/blob/main/docs/operations.md#rotate-the-tunnel-token).
How it fits in the stack is in [architecture](architecture.md#remote-access), and the route and Access application for n8n are in the [setup guide](setup.md#cloudflare-tunnel-and-access).

### When lab-n8n.jotaeme.dev does not load

Go down the list until one check fails.

1. **n8n runs.** `docker compose ps` shows `n8n` and `n8n-runners` as `Up`.
2. **n8n answers.** `curl -s -o /dev/null -w '%{http_code}' http://localhost:5678/healthz` prints `200`.
3. **The tunnel, DNS and Access work.** Follow [When a hostname does not load](https://github.com/JuanMaVelezPa/jotaeme-ingress/blob/main/docs/operations.md#when-a-hostname-does-not-load) in jotaeme-ingress.
   If the PIN email does not arrive, check the spam folder and the email in the `Owner` policy of the `lab-n8n` Access application.
4. **`cloudflared` reaches n8n.** A `502` page after the PIN means `cloudflared` cannot reach `http://n8n:5678`.
   Check that both `n8n` and `cloudflared` are on the `edge` network with `docker network inspect edge`, and that the tunnel route points to `http://n8n:5678`.

While the domain is down, the workflows keep running on schedule, and the editor is still at http://localhost:5678 on the server.
Devices on the LAN cannot use `http://<server IP>:5678`, because n8n listens only on `127.0.0.1`. They use the domain.

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
