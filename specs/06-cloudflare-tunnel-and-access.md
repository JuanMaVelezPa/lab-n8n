# SPEC 06 — Cloudflare Tunnel and Access for n8n

> **Status:** Implemented
> **Depends on:** SPEC 05
> **Date:** 2026-10-27
> **Objective:** Publish the n8n editor at `https://lab-n8n.jotaeme.dev` through a Cloudflare Tunnel run by a `cloudflared` container in this compose, behind Cloudflare Access with a one-time PIN for one email, and document it in the shared guides.

## Why this spec exists

SPEC 05 split the user's request in two: documentation first, remote access second.
Today n8n is only reachable at `http://localhost:5678` on the server, and at port `5678` from any device on the LAN with no protection other than the n8n login.
The domain `jotaeme.dev` is already managed in Cloudflare, and SPEC 05 fixed the naming `lab-<name>.jotaeme.dev`, so n8n becomes `lab-n8n.jotaeme.dev`.
SPEC 05 also kept the words `tunnel` and `Cloudflare` out of the docs on purpose, so this spec is the one that adds them.

## Scope

**In:**

- A `cloudflared` service in `docker-compose.yml`, image `cloudflare/cloudflared:latest`, on the `ia` network, started with the token in `.env`.
- A remotely-managed tunnel `lab-n8n`, created in the Cloudflare Zero Trust dashboard, with one public hostname: `lab-n8n.jotaeme.dev` → `http://n8n:5678`.
- One Cloudflare Access self-hosted application for the whole hostname `lab-n8n.jotaeme.dev`, with one Allow policy for one email address, the One-time PIN login and a 24-hour session.
- n8n bound to `127.0.0.1:5678` instead of every interface.
- n8n environment for the public HTTPS hostname: `N8N_HOST`, `N8N_PROTOCOL`, `N8N_EDITOR_BASE_URL`, `WEBHOOK_URL`, `N8N_PROXY_HOPS` and `N8N_SECURE_COOKIE=true`.
- A second authorized redirect URI in the Google OAuth client, for the domain. The `localhost` URI stays.
- A committed `.env.example` and a `!.env.example` exception in `.gitignore`.
- Updates to `docs/setup.md`, `docs/architecture.md`, `docs/operations.md` and `README.md` to describe the tunnel and Access.

**Out of scope (for future specs):**

- Other `lab-*.jotaeme.dev` subdomains or other apps behind the tunnel.
- Moving `cloudflared` to a shared stack outside this repo. That waits until a second lab app needs the tunnel.
- Cloudflare WAF, rate limiting or any rule other than the Access application.
- An Access bypass for `/webhook/*` or `/webhook-test/*`. No workflow uses webhooks today. The spec that adds one decides its bypass.
- Google or any other identity provider in Access.
- n8n MFA, extra n8n users or changes to the n8n owner login.
- Managing Cloudflare with Terraform, the API or a local `config.yml`.
- Pinning the `cloudflared` or n8n image to a version.
- Any change to workflow logic, workflow files, sheet templates or specs 01 to 05.

## Data model

This feature introduces no new runtime data in n8n or in the sheets. It adds one container, one secret and configuration in Cloudflare and Google Cloud.

**Files**

| Path | Change |
| --- | --- |
| `docker-compose.yml` | New `cloudflared` service. n8n port bound to `127.0.0.1`. New n8n environment variables. |
| `.env` (not committed) | New `TUNNEL_TOKEN`. |
| `.env.example` | New. Committed. |
| `.gitignore` | New line `!.env.example`. |
| `docs/setup.md` | New Cloudflare Tunnel and Access section. Both redirect URIs. |
| `docs/architecture.md` | `cloudflared` in the containers table and the stack diagram. Access in front of n8n. |
| `docs/operations.md` | Tunnel logs and status, token rotation, troubleshooting when the domain does not load. |
| `README.md` | Architecture diagram, quick start and requirements mention the tunnel and Access. |

**`docker-compose.yml`** (target shape, not final text)

```yaml
services:
  n8n:
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - GENERIC_TIMEZONE=America/Bogota
      - TZ=America/Bogota
      - N8N_HOST=lab-n8n.jotaeme.dev
      - N8N_PROTOCOL=https
      - N8N_EDITOR_BASE_URL=https://lab-n8n.jotaeme.dev/
      - WEBHOOK_URL=https://lab-n8n.jotaeme.dev/
      - N8N_PROXY_HOPS=1
      - N8N_SECURE_COOKIE=true

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    depends_on:
      - n8n
    networks:
      - ia
```

**`.env.example`**

```text
COMPOSE_PROJECT_NAME=n8n
TUNNEL_TOKEN=<tunnel token from Cloudflare Zero Trust>
```

**Cloudflare configuration** (dashboard, recorded here so it can be repeated)

```text
Tunnel
  name:             lab-n8n
  type:             Cloudflared, remotely managed
  public hostname:  lab-n8n.jotaeme.dev
  service:          HTTP  n8n:5678

Access application
  type:             Self-hosted
  name:             lab-n8n
  domain:           lab-n8n.jotaeme.dev   (no path, so the whole hostname)
  session duration: 24 hours
  login methods:    One-time PIN only
  policy:           name "Owner", action Allow, include Emails = <owner email>
```

**Google OAuth client, authorized redirect URIs**

```text
http://localhost:5678/rest/oauth2-credential/callback      (kept)
https://lab-n8n.jotaeme.dev/rest/oauth2-credential/callback (new)
```

Conventions:

- The tunnel token lives only in `.env` and in Cloudflare. It never appears in the repo, the docs or a commit.
- Docs use `you@example.com` for the Access email and `<tunnel token>` for the token, as SPEC 05 set.
- Docs keep one sentence per line and relative links, as SPEC 05 set.
- Steps in the Cloudflare and Google dashboards are done by the user. Steps on the server and in the repo are done by the agent.

## Implementation plan

1. Add `.env.example` and the `!.env.example` line in `.gitignore`. Check: `git status` shows `.env.example` as a new file and `git check-ignore .env` still matches.
2. The user creates the Access application in **Zero Trust → Access → Applications** with the values in the Data model, before the hostname exists. Check: the application is listed with the One-time PIN login and the `Owner` policy.
3. The user creates the tunnel `lab-n8n` in **Zero Trust → Networks → Tunnels**, adds the public hostname `lab-n8n.jotaeme.dev` → `HTTP n8n:5678`, and puts the token in `.env` as `TUNNEL_TOKEN`. Check: **DNS** in the `jotaeme.dev` zone shows a proxied `CNAME lab-n8n` to `<tunnel id>.cfargotunnel.com`.
4. Add the `cloudflared` service to `docker-compose.yml` and run `docker compose up -d cloudflared`. Check: `docker logs cloudflared` shows `Registered tunnel connection`, and the tunnel shows `HEALTHY` in the dashboard.
5. Change the n8n port to `127.0.0.1:5678:5678`, add the n8n environment variables from the Data model and run `docker compose up -d n8n`. Check: `curl -s -o /dev/null -w '%{http_code}' http://localhost:5678/healthz` returns `200`, and the same request to `http://<server LAN IP>:5678/healthz` fails to connect.
6. The user adds the domain redirect URI to the Google OAuth client in **Google Cloud → APIs & Services → Credentials**. Check: the client lists both URIs, and in n8n both credentials still show as connected.
7. Update `docs/setup.md`: a new `### Cloudflare Tunnel and Access` subsection after the n8n start, with the tunnel and Access, both redirect URIs in the OAuth step, and the "open n8n at `http://localhost:5678`" note rewritten for both hosts. The n8n start copies `.env.example` to `.env` before the first `docker compose up -d n8n`, so the volume is `n8n_n8n_data` from the start and `cloudflared` does not start without a token. The subsection is not numbered, so the `#5-…` and `#6-…` anchors linked from `projects/` keep working. Check: the steps can be followed in order from a clean server.
8. Update `docs/architecture.md`: `cloudflared` in the containers table, the n8n row with `127.0.0.1:5678`, and the stack diagram with the path browser → Cloudflare Access → tunnel → `cloudflared` → n8n. Check: the diagram renders in the GitHub preview of the branch.
9. Update `docs/operations.md`: `docker logs cloudflared`, how to rotate the token (refresh it in the dashboard, update `.env`, `docker compose up -d cloudflared`), and what to check when `lab-n8n.jotaeme.dev` does not load. Note that `docker compose pull` also updates `cloudflared`. Check: every command runs from the repo root.
10. Update `README.md`: the architecture diagram with Cloudflare Access and `cloudflared`, a line in the quick start about `.env.example`, and the Cloudflare account and domain in the requirements. Check: the README renders on GitHub and every link resolves.

## Acceptance criteria

- [x] `docker compose ps` shows `n8n` and `cloudflared` running.
- [x] `docker logs cloudflared` contains `Registered tunnel connection`.
- [x] The tunnel `lab-n8n` shows `HEALTHY` in Cloudflare Zero Trust.
- [x] `curl -sI https://lab-n8n.jotaeme.dev` from outside the LAN returns a redirect to a `cloudflareaccess.com` login page.
- [x] Opening `https://lab-n8n.jotaeme.dev` in a private window asks for an email, sends a PIN to the owner email, and after the PIN shows the n8n sign-in page.
- [ ] Another email address entered on the Access page does not get to the n8n sign-in page.
- [x] After signing in to n8n through the domain, the editor loads and both workflows are listed and published.
- [x] `curl -s -o /dev/null -w '%{http_code}' http://localhost:5678/healthz` on the server returns `200`.
- [x] A request to `http://<server LAN IP>:5678` from another device on the LAN fails to connect.
- [x] Signing in to n8n at `http://localhost:5678` on the server still works.
- [x] In n8n, the OAuth redirect URL shown in the `Google Sheets account` and `Gmail account` credentials is `https://lab-n8n.jotaeme.dev/rest/oauth2-credential/callback`.
- [x] The Google OAuth client lists both redirect URIs from the Data model.
- [x] The next scheduled run of each workflow sends its email, with no failed execution in **Executions**.
- [x] `.env.example` is committed, and `.env` is not.
- [x] `git grep -n TUNNEL_TOKEN` shows only `docker-compose.yml`, `.env.example`, `docs/*.md` and this spec, with no token value.
- [x] `docs/setup.md`, `docs/architecture.md`, `docs/operations.md` and `README.md` mention `lab-n8n.jotaeme.dev` and Cloudflare Access.
- [x] Every Mermaid diagram renders on the GitHub page, with no "Unable to render" error.
- [x] Every relative link in `README.md` and `docs/*.md` points to a file that exists, and every `#anchor` matches a heading.
- [x] `git grep -nE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-z]{2,}' -- '*.md'` finds only `example.com` and `anthropic.com` addresses.
- [x] No file under `projects/`, `specs/01-*` to `specs/05-*` is changed.

## Decisions

- **Yes:** `cloudflared` as a service in this repo's `docker-compose.yml`. Chosen by the user. Self-contained and versioned with the repo, and it reaches n8n by name on the `ia` network.
- **No:** a shared `cloudflared` stack outside the repo. Deferred until a second lab app needs the tunnel.
- **No:** `cloudflared` as a host systemd service. Not in Docker and not versioned.
- **Yes:** remotely-managed tunnel with a token. Chosen by the user. Only one secret on the server, and the spec records the dashboard values so they can be repeated.
- **No:** a locally-managed tunnel with `config.yml` in the repo. Needs the `cloudflared tunnel login` certificate and a credentials file for one hostname.
- **Yes:** Cloudflare Access with the One-time PIN login, one allowed email and a 24-hour session. Chosen by the user. No identity provider to set up, and the Cloudflare default session.
- **No:** Google as identity provider. Needs another OAuth client for a one-person lab.
- **Yes:** keep the n8n owner login behind Access. Two layers: Access stops everyone else before n8n, and n8n still asks for its own password.
- **Yes:** Access covers the whole hostname, webhooks included. Chosen by the user. No workflow uses webhooks today.
- **No:** a `/webhook/*` bypass now. It would open a path that nothing uses.
- **Yes:** create the Access application before the public hostname. The hostname is never public without Access in front.
- **Yes:** bind n8n to `127.0.0.1:5678`. Chosen by the user. LAN devices lose direct access, which had only the n8n login in front, and the server keeps `localhost` for local use and the OAuth fallback.
- **No:** keep `0.0.0.0:5678`. It would leave a path to n8n that skips Access.
- **No:** remove the port. Local use and the `localhost` redirect URI would stop working if the tunnel is down.
- **Yes:** set `N8N_HOST`, `N8N_PROTOCOL`, `N8N_EDITOR_BASE_URL`, `WEBHOOK_URL`, `N8N_PROXY_HOPS=1` and `N8N_SECURE_COOKIE=true`. Chosen by the user. n8n builds its OAuth and webhook URLs with the public HTTPS hostname and trusts the proxy headers.
- **Yes:** set `N8N_EDITOR_BASE_URL` too. Without it n8n adds the port and builds `https://lab-n8n.jotaeme.dev:5678/` URLs.
- **No:** keep `N8N_SECURE_COOKIE=false`. The editor is now served over HTTPS. Browsers still accept secure cookies on `http://localhost`.
- **Yes:** add the domain redirect URI and keep the `localhost` one. Chosen by the user. Existing tokens keep working, and reconnecting works through the domain.
- **No:** replace the `localhost` redirect URI. It would remove the fallback when the tunnel is down.
- **Yes:** `.env.example` committed with a `!.env.example` exception. Chosen by the user. Setup can copy it, and it holds no secret.
- **Yes:** image `cloudflare/cloudflared:latest`. Chosen by the user. Same policy as n8n, updated by the same `docker compose pull`.
- **Yes:** update `docs/setup.md`, `docs/architecture.md`, `docs/operations.md` and `README.md`. Recommended and chosen by the user. SPEC 05 left the mentions of remote access to this spec.
- **No:** WAF or rate limit rules, a shared stack and n8n MFA in this spec. Chosen by the user. Each one is its own area.
- **Yes:** Cloudflare and Google dashboard steps done by the user. The agent has no access to those accounts.

## Risks

| Risk | Mitigation |
| --- | --- |
| The hostname is public for a moment without Access. | Step 2 creates the Access application before step 3 creates the hostname. |
| The tunnel token leaks into a commit. | It lives only in `.env`, which is ignored. The `TUNNEL_TOKEN` grep criterion checks the repo. If it leaks, rotate it as in `docs/operations.md`. |
| `N8N_SECURE_COOKIE=true` breaks the login at `http://localhost:5678` in some browser. | Chrome and Firefox treat `localhost` as secure. If it fails, the domain still works. The acceptance criterion checks the local login. |
| A Google credential reconnect fails with `redirect_uri_mismatch`. | Step 6 adds the domain URI before anyone reconnects. The criterion checks the URI n8n shows. |
| The tunnel or Cloudflare is down. | The workflows run inside n8n on a schedule and do not need the tunnel. Local access at `localhost` keeps working. |
| `cloudflared:latest` brings a breaking change on `docker compose pull`. | `docs/operations.md` says to check `docker logs cloudflared` after an update. Pinning stays out of scope. |
| A LAN device or script used `http://<server IP>:5678`. | It must use `https://lab-n8n.jotaeme.dev`. The docs say so. |
| The Access PIN email does not arrive. | Local access at `localhost` keeps working. The Access policy can be checked in the dashboard. |

## What is **not** in this spec

- Other `lab-*.jotaeme.dev` subdomains or apps.
- A shared `cloudflared` stack.
- WAF, rate limiting or other Cloudflare rules.
- Webhook bypass in Access.
- Google or other identity providers.
- n8n MFA or user changes.
- Terraform, API or local config for Cloudflare.
- Pinned image versions.
- Changes to workflows, sheet templates or earlier specs.

Each one of those, if it lands, goes in its own spec.
