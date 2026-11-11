# SPEC 09 — Move lab-n8n to the edge and llm networks

> **Status:** Implemented
> **Depends on:** SPEC 08, jotaeme-ingress SPEC 01
> **Date:** 2026-11-10
> **Objective:** Rename the Compose project to `lab-n8n`, remove `cloudflared` and the `ia` network from this repo, put `n8n` on `default` + `edge` + `llm` and `n8n-runners` on `default` only, and keep the existing `n8n_n8n_data` volume, applied in step 6.2 of the jotaeme-ingress SPEC 01 cutover.

## Why this spec exists

jotaeme-ingress SPEC 01 moves `cloudflared` into its own stack and replaces the catch-all `ia` network with role-based networks.
Its scope lists this spec as the lab-n8n half of that move, next to `jotaeme-llm` SPEC 01 for Ollama.
The order of work across the three repos, and the full rollback, are in the "Coordinated runbook" of jotaeme-ingress SPEC 01.
This spec is row 3 (prepare), row 5 (`.env`), row 7 (cutover step 6.2) and row 12 (pull request) of that runbook.
The target topology from that spec is:

```text
cloudflared   → edge                      (jotaeme-ingress)
n8n           → lab-n8n_default + edge + llm
n8n-runners   → lab-n8n_default
ollama        → llm                       (jotaeme-llm)
```

Two facts found while writing this spec shape the plan:

- The volume `n8n_n8n_data` carries the label `com.docker.compose.project=n8n`. Under the project `lab-n8n`, Compose would create a new, empty `lab-n8n_n8n_data` unless the volume is declared with `name: n8n_n8n_data`.
- The running containers `n8n`, `n8n-runners` and `cloudflared` belong to the Compose project `n8n`. `--remove-orphans` only removes orphans of the current project, so `docker compose up -d --remove-orphans` under `lab-n8n` does not see them. It fails with `container name "/n8n" is already in use`, and `cloudflared` stays. Step 6.2 of jotaeme-ingress SPEC 01 takes the old project down first. That step was corrected on 2026-11-10, while the three specs were aligned.

## Scope

**In:**

- `COMPOSE_PROJECT_NAME=lab-n8n` in `.env.example` and in the local `.env`.
- The `n8n_data` volume declared with `name: n8n_n8n_data`, so n8n keeps the current volume.
- The `cloudflared` service removed from `docker-compose.yml`, and `TUNNEL_TOKEN` removed from `.env.example` and `.env`.
- `n8n` on the networks `default`, `edge` and `llm`.
- `n8n-runners` on the network `default` only.
- The `ia` network removed from `docker-compose.yml`. `edge` and `llm` declared as external.
- Docs updated: `README.md` (diagram, text, Quick start, requirements, layout), `docs/architecture.md`, `docs/setup.md` and `docs/operations.md`.
- The tunnel and Ollama setup steps replaced by links to `jotaeme-ingress` and `jotaeme-llm`, keeping what belongs to n8n: the route `lab-n8n.jotaeme.dev → http://n8n:5678`, the Access application, the `N8N_*` variables and the `502` troubleshooting.
- A fresh backup of `n8n_n8n_data` during the cutover, after the old containers stop and before the new ones start.

**Out of scope (for future specs):**

- Creating the `edge` and `llm` networks on the host, the `cloudflared` stack, the tunnel rename and `docker network rm ia`. They are steps of jotaeme-ingress SPEC 01.
- Moving Ollama to `llm` and its own repo. That is jotaeme-llm SPEC 01.
- Any change to workflows, workflow files in `projects/*/workflows/`, sheet templates or credentials. The internal URLs `http://ollama:11434` and `http://n8n:5679` do not change.
- Renaming the volume to `lab-n8n_n8n_data` or migrating its data to a new volume.
- Changing the n8n or runners image tags.
- Changing the host port `127.0.0.1:5678` or any `N8N_*` variable.
- Applying the change on its own, outside the jotaeme-ingress cutover.

## Data model

This feature introduces no new runtime data.
It changes the Compose project, networks and services.

**Files**

| Path | Change |
| --- | --- |
| `docker-compose.yml` | `cloudflared` removed. `n8n` on `default`, `edge`, `llm`. `n8n-runners` on `default`. Volume `name: n8n_n8n_data`. Networks `edge` and `llm` external, `ia` removed. |
| `.env.example` | `COMPOSE_PROJECT_NAME=lab-n8n`. `TUNNEL_TOKEN` line removed. |
| `.env` (not committed) | `COMPOSE_PROJECT_NAME=lab-n8n`. `TUNNEL_TOKEN` line removed, after the user has copied it to jotaeme-ingress (its step 5). |
| `.env.pre-spec09` (not committed, ignored by `.env.*`) | Copy of the old `.env`, for the full rollback. Deleted after jotaeme-ingress step 8. |
| `README.md` | Diagram, architecture text, Quick start, requirements and repository layout. |
| `docs/architecture.md` | Containers table, network text and diagram. |
| `docs/setup.md` | Networks, Ollama, `.env`, tunnel and Access sections. |
| `docs/operations.md` | Backup text about the volume name, "Update n8n", "Cloudflare Tunnel" and troubleshooting. |
| `backups/n8n_data-<date>-cutover.tar.gz` (not committed) | Backup taken during the cutover. |

**`docker-compose.yml`** (target shape, not final text)

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:3.2.0
    container_name: n8n
    # ...ports, environment and volumes unchanged...
    networks:
      - default   # private floor: n8n-runners reaches the broker at n8n:5679
      - edge      # lobby: cloudflared reaches n8n:5678
      - llm       # n8n calls http://ollama:11434

  n8n-runners:
    image: n8nio/runners:3.2.0
    container_name: n8n-runners
    # ...environment and depends_on unchanged...
    networks:
      - default

volumes:
  n8n_data:
    # Keeps the volume created under the old project name `n8n`.
    # Never remove this line: without it Compose creates an empty lab-n8n_n8n_data.
    name: n8n_n8n_data

networks:
  # Created once on the host by jotaeme-ingress.
  edge:
    external: true
  llm:
    external: true
```

The volume is not `external`, so a fresh install still gets it from `docker compose up`.
On this server Compose prints a warning that the volume was created for project `n8n`. The warning is expected and harmless.

**`.env.example`**

```text
COMPOSE_PROJECT_NAME=lab-n8n
N8N_RUNNERS_AUTH_TOKEN=<random hex string, from openssl rand -hex 32>
```

**README Quick start** (target content, not final text)

```bash
docker network create edge   # skip if jotaeme-ingress already created it
docker network create llm    # skip if jotaeme-ingress already created it
cp .env.example .env
sed -i "s|^N8N_RUNNERS_AUTH_TOKEN=.*|N8N_RUNNERS_AUTH_TOKEN=$(openssl rand -hex 32)|" .env
docker compose up -d
```

A note says an `already exists` error from `docker network create` can be ignored.
`docker compose up -d` no longer names services, because `cloudflared` is gone.

**Docs content** (target content, not final text)

- Diagrams in `README.md` and `docs/architecture.md`: `cloudflared` labelled as the jotaeme-ingress container, `cloudflared → n8n:5678` on `edge`, `n8n-runners → n8n:5679` on `default`, `n8n → ollama:11434` on `llm`.
- `docs/architecture.md`: the containers table keeps `cloudflared` and Ollama as rows owned by `jotaeme-ingress` and `jotaeme-llm`. The network text explains the three networks and that the broker port 5679 is reachable only on `lab-n8n_default`.
- `docs/setup.md`:
  - "Create the Docker network" becomes "Create the Docker networks", with `edge` and `llm`.
  - The Ollama section links to `jotaeme-llm` instead of showing a compose file.
  - The `.env` text says `COMPOSE_PROJECT_NAME=lab-n8n`, and that the volume is still `n8n_n8n_data` because of `name:` in `docker-compose.yml`.
  - The tunnel section links to `jotaeme-ingress` for the tunnel and `cloudflared`, and keeps the route to `http://n8n:5678` and the Access application.
  - The Ollama check runs over the `llm` network.
- `docs/operations.md`:
  - The volume text says the name is fixed to `n8n_n8n_data` in `docker-compose.yml`.
  - "Update n8n" drops the note about not touching `cloudflared`.
  - "Cloudflare Tunnel" becomes a short pointer to `jotaeme-ingress/docs/operations.md`, including token rotation.
  - Troubleshooting checks `n8n` and `n8n-runners` with `docker compose ps`, and the `502` case checks `docker network inspect edge`.

**Cutover commands** (step 8 below, the same text as step 6.2 of jotaeme-ingress SPEC 01)

```bash
docker compose -p n8n down        # old project: removes n8n, n8n-runners, cloudflared. Never add -v.
docker run --rm -v n8n_n8n_data:/data -v "$PWD/backups:/backup" alpine \
  tar czf /backup/n8n_data-$(date +%F)-cutover.tar.gz -C /data .
docker compose up -d --remove-orphans   # new project lab-n8n
```

Conventions:

- The Compose project is `lab-n8n`, so the private network is `lab-n8n_default`.
- The volume name `n8n_n8n_data` is fixed in `docker-compose.yml`, not derived from the project name.
- Between implementation and the cutover, no `docker compose` command runs in lab-n8n without `-p n8n`.

## Implementation plan

Steps marked *(user)* are run by the user; Claude proposes the exact commands.

1. In `docker-compose.yml`: remove `cloudflared`, set the networks of `n8n` and `n8n-runners`, add `name: n8n_n8n_data` to the volume, and replace `ia` with external `edge` and `llm`. Check: `COMPOSE_PROJECT_NAME=lab-n8n N8N_RUNNERS_AUTH_TOKEN=x docker compose config` is valid, lists only `n8n` and `n8n-runners`, and shows the volume `name: n8n_n8n_data`. Nothing is started.
2. Update `.env.example`: `COMPOSE_PROJECT_NAME=lab-n8n` and no `TUNNEL_TOKEN`. Check: `cat .env.example` shows two lines.
3. Update `README.md` as in the Data model. Check: `grep -n '\bia\b\|cloudflared container\|TUNNEL_TOKEN' README.md` finds no stale reference.
4. Update `docs/architecture.md`. Check: no mention of the `ia` network, and the diagram renders on GitHub.
5. Update `docs/setup.md`. Check: no `docker network create ia`, no Ollama compose file, no `TUNNEL_TOKEN`, and `COMPOSE_PROJECT_NAME=lab-n8n`.
6. Update `docs/operations.md`. Check: no `docker compose up -d cloudflared`, no `docker network inspect ia`.
7. *(user)* After jotaeme-ingress step 5 (token copied there): `cp .env .env.pre-spec09`, then set `COMPOSE_PROJECT_NAME=lab-n8n` in `.env` and remove `TUNNEL_TOKEN`. Check: `grep -c TUNNEL_TOKEN .env` prints `0`, and `git status` does not list `.env.pre-spec09`. Do not run `docker compose` in lab-n8n after this, until step 8.
8. *(user)* Cutover, as jotaeme-ingress step 6.2, after 6.1 passed and with `edge` and `llm` created. Run the three cutover commands from the Data model, from the lab-n8n folder on this branch. Check:
   - `ls -lh backups/` shows `n8n_data-<date>-cutover.tar.gz`, larger than 10 MB.
   - `docker volume ls` lists `n8n_n8n_data` and no `lab-n8n_n8n_data`.
   - `docker ps` shows `n8n` and `n8n-runners` with compose project `lab-n8n`, and no `cloudflared` from project `n8n`.
   - `docker inspect -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{end}}' n8n` prints `edge lab-n8n_default llm`, and the same for `n8n-runners` prints `lab-n8n_default`.
   - `docker logs n8n-runners` shows the launcher connected to the broker.
   - `docker exec n8n wget -qO- http://ollama:11434/api/version` prints the Ollama version.
   - At http://localhost:5678, after sign-in: `daily-gospel` and `news-digest` are listed and published, the `Google Sheets account` and `Gmail account` credentials exist, and **Executions** still shows the past runs.
9. After jotaeme-ingress step 6.3 and its checks pass, open the pull request for this branch. The merge happens after the cutover, so `main` never documents a state that is not running. After jotaeme-ingress step 8, delete `.env.pre-spec09`.

If step 8 fails, use the full rollback in the "Coordinated runbook" of jotaeme-ingress SPEC 01: `docker compose down` here (never `-v`), restore `docker-compose.yml` from `main` and `.env` from `.env.pre-spec09`, run `docker compose -p n8n up -d`, and roll Ollama back too, because the old n8n reaches Ollama only on `ia`.
If the volume looks empty after `up`, stop n8n at once, check `docker volume ls` for a new `lab-n8n_n8n_data`, and fix the `name:` line. The data in `n8n_n8n_data` is untouched, and the cutover backup is the fallback.

## Acceptance criteria

- [x] `docker-compose.yml` has no `cloudflared` service and no `ia` network.
- [x] `docker-compose.yml` declares the volume `n8n_data` with `name: n8n_n8n_data`, and `edge` and `llm` as external networks.
- [x] `.env.example` sets `COMPOSE_PROJECT_NAME=lab-n8n` and has no `TUNNEL_TOKEN`.
- [x] `.env` sets `COMPOSE_PROJECT_NAME=lab-n8n` and has no `TUNNEL_TOKEN`.
- [x] `docker volume ls` lists `n8n_n8n_data` and no `lab-n8n_n8n_data`.
- [x] `n8n` is on `lab-n8n_default`, `edge` and `llm`, and on no other network.
- [x] `n8n-runners` is only on `lab-n8n_default`.
- [x] No container of compose project `n8n` exists.
- [x] After the cutover, `daily-gospel` and `news-digest` are listed and published, and both credentials exist in **Credentials**.
- [x] **Executions** shows runs from before the cutover.
- [x] `docker exec n8n wget -qO- http://ollama:11434/api/version` prints the Ollama version.
- [x] `backups/` holds `n8n_data-<date>-cutover.tar.gz` larger than 10 MB.
- [x] `git grep -n '\bia\b' -- README.md docs/` finds no reference to the `ia` network, and `git grep -n TUNNEL_TOKEN` finds nothing outside `specs/`.
- [x] `README.md` Quick start creates `edge` and `llm`, not `ia`, and runs `docker compose up -d`.
- [x] `docs/setup.md` links to `jotaeme-ingress` for the tunnel and to `jotaeme-llm` for Ollama, and still documents the route to `http://n8n:5678` and the Access application.
- [x] Step 6.2 of jotaeme-ingress SPEC 01 runs `docker compose -p n8n down`, the backup and `docker compose up -d --remove-orphans`. Done on 2026-11-10, while the three specs were aligned.
- [x] No file in `projects/*/workflows/` changed: `git diff main --stat -- projects` is empty.

## Decisions

- **Yes:** project name `lab-n8n`, matching the folder and the `lab-n8n_default` network in jotaeme-ingress SPEC 01. Chosen by the user.
- **Yes:** keep the volume with `name: n8n_n8n_data`. Chosen by the user. Workflows, credentials, the encryption key and executions stay where they are, with no copy.
- **No:** `external: true` on the volume. It would hide the Compose warning, but a fresh install would need `docker volume create` before the first start.
- **No:** copy the data into a new `lab-n8n_n8n_data`. More steps and more risk for a cosmetic name.
- **Yes:** take the old project down with `docker compose -p n8n down`, then `docker compose up -d --remove-orphans`. Chosen by the user. `--remove-orphans` alone cannot see containers of another project, and the fixed `container_name` values would clash.
- **No:** `docker rm -f n8n n8n-runners cloudflared`. It works, but bypasses Compose.
- **No:** keep `COMPOSE_PROJECT_NAME=n8n` so `--remove-orphans` works as written. Not selected.
- **Yes:** a fresh backup during the cutover, by volume name, while no container runs. Chosen by the user. The containers are removed at that point, so `--volumes-from n8n` from `docs/operations.md` cannot be used.
- **Yes:** fix step 6.2 in jotaeme-ingress SPEC 01. Chosen by the user. The cutover is run from that spec, and the old text would fail. Done on 2026-11-10 together with a coordinated runbook for the three specs.
- **Yes:** keep the old `.env` as `.env.pre-spec09` until `ia` is removed. The full rollback needs the old project name and `TUNNEL_TOKEN`.
- **Yes:** the tunnel and Ollama setup steps move out to links to `jotaeme-ingress` and `jotaeme-llm`, and the n8n-specific parts stay here. Chosen by the user. One source of truth per stack.
- **No:** keep the full tunnel steps adapted, or delete everything about the tunnel. The first duplicates jotaeme-ingress, the second loses the route and Access app that only matter to n8n.
- **Yes:** the Quick start creates `edge` and `llm` itself, with a note that `already exists` is fine. Chosen by the user. A visitor can start n8n without installing jotaeme-ingress first.
- **No:** make jotaeme-ingress a prerequisite of the Quick start. Not selected.
- **Yes:** `n8n-runners` only on `default`. The runners need only the broker, and leave `llm` and `edge`, which is least privilege.
- **Yes:** apply only in jotaeme-ingress step 6.2, never on its own. Chosen by the user. Without `edge`, `cloudflared` from jotaeme-ingress cannot reach n8n, and without `llm` n8n cannot reach Ollama.
- **Yes:** open the pull request after the cutover checks pass. `main` keeps describing the running stack until the switch happens.

## Risks

| Risk | Mitigation |
| --- | --- |
| The `name:` line is missing or wrong, and n8n starts on an empty new volume. | Step 1 checks `docker compose config`. Step 8 checks `docker volume ls` and the workflow list. The old volume is never touched, and the cutover backup is the fallback. |
| Someone runs `docker compose down -v` in lab-n8n. | `-v` would delete `n8n_n8n_data`, now that it belongs to this compose file. The compose comment and `docs/operations.md` warn against it, and backups exist. |
| A `docker compose` command runs in lab-n8n between step 7 and the cutover. | It targets the empty project `lab-n8n`. `up` fails on the container name clash, so nothing changes. Step 7 says to run nothing until step 8. |
| `edge` or `llm` does not exist at step 8. | `up` fails with `network edge declared as external, but could not be found`. Create it (jotaeme-ingress step 4) and run `up` again. |
| A scheduled workflow runs during the cutover. | Do it outside the workflow schedules, as jotaeme-ingress SPEC 01 says. |
| The links to `jotaeme-ingress` and `jotaeme-llm` point to repos not yet public on GitHub. | Those repos are published as part of their own specs. The links work once they are pushed. |
| The Compose warning about the volume's project label is read as an error. | `docs/operations.md` and the compose comment say it is expected. |

## What is **not** in this spec

- Creating `edge` or `llm`, the `cloudflared` stack, the tunnel rename and removing the `ia` network from the host.
- Moving Ollama.
- Workflow, sheet or credential changes.
- Renaming or migrating the volume.
- Image tag, port or `N8N_*` changes.
- Applying the change outside the jotaeme-ingress cutover.

Each one of those, if it lands, goes in its own spec.
