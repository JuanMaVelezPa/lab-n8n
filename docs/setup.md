# Setup

The shared setup for every project in this repo: Docker, the Docker networks, Ollama, n8n, the public hostname with Access, and the Google credentials.
Do it once, in this order. Then follow the README of each project.

Commands are run from the repo root unless a step says otherwise.

## 1. Install Docker

Install Docker Engine and the Compose plugin by following https://docs.docker.com/engine/install/.

Check: both commands print a version.

```bash
docker --version
docker compose version
```

## 2. Create the Docker networks

n8n joins two shared Docker networks, created once on the host:

- `edge`: `cloudflared` from [jotaeme-ingress](https://github.com/JuanMaVelezPa/jotaeme-ingress) reaches n8n here.
- `llm`: n8n reaches Ollama from [jotaeme-llm](https://github.com/JuanMaVelezPa/jotaeme-llm) here.

If jotaeme-ingress is already installed, both exist. Otherwise create them:

```bash
docker network create edge
docker network create llm
```

An `already exists` error means the network is there, and can be ignored.
The third network, `lab-n8n_default`, is created by Compose and links n8n with `n8n-runners`.

Check: `docker network ls` lists `edge` and `llm`.

## 3. Start Ollama

Ollama runs in its own container, named `ollama`, on the `llm` network.
It lives in its own repo: follow the [jotaeme-llm Quick start](https://github.com/JuanMaVelezPa/jotaeme-llm#quick-start), which lists its own requirements, including the NVIDIA Container Toolkit for the GPU.
The project templates use the model `gemma4:12b`:

```bash
docker exec ollama ollama pull gemma4:12b
```

Check: `docker exec ollama ollama list` shows `gemma4:12b`.

## 4. Start n8n

From the repo root, create `.env` from the example before the first start:

```bash
cp .env.example .env
sed -i "s|^N8N_RUNNERS_AUTH_TOKEN=.*|N8N_RUNNERS_AUTH_TOKEN=$(openssl rand -hex 32)|" .env
docker compose up -d
```

`.env` sets `COMPOSE_PROJECT_NAME=lab-n8n`, so the private network is `lab-n8n_default`, whatever the folder is called.
The data volume is still `n8n_n8n_data`, the name from the old project `n8n`, because `docker-compose.yml` fixes it with `name: n8n_n8n_data`.
The `sed` line gives `N8N_RUNNERS_AUTH_TOKEN` a random value. n8n and `n8n-runners` share it, and `n8n-runners` runs the code of the Code nodes.
`.env` is ignored by git.

Open http://localhost:5678 and create the owner account.
n8n listens only on `127.0.0.1:5678`, so `localhost` works on the server and other devices use the domain below.

Check: n8n can reach Ollama over the `llm` network.

```bash
docker exec n8n wget -qO- http://ollama:11434/api/version
```

It prints the Ollama version as JSON.

### Cloudflare Tunnel and Access

n8n is published at `https://lab-n8n.jotaeme.dev` through a Cloudflare Tunnel.
The tunnel and its `cloudflared` container live in [jotaeme-ingress](https://github.com/JuanMaVelezPa/jotaeme-ingress), which reaches n8n by name on the `edge` network, so no port is opened on the router.
Cloudflare Access stands in front of the whole hostname and asks for a one-time PIN sent by email before anyone reaches n8n.
n8n still asks for its own login after that.

Install jotaeme-ingress first, by its own README. Then do the two parts that belong to n8n, by hand and in this order, so the hostname is never public without Access.
You need a Cloudflare account with the `jotaeme.dev` zone, and Zero Trust enabled on the Free plan.

**1. Turn on the one-time PIN login.**
In **Zero Trust → Integrations → Identity providers** (or **Settings → Authentication**), add **One-time PIN** if it is not listed.

**2. Create the Access application.**
In **Zero Trust → Access → Applications**, add an application of type **Self-hosted and private**:

- **Public hostname:** subdomain `lab-n8n`, domain `jotaeme.dev`, empty path.
- **Policy:** name `Owner`, action **Allow**, include **Emails** `you@example.com`.
- **Authentication:** only **One-time PIN**, with **Apply instant authentication** on.
- **Details:** name `lab-n8n`, session duration **24 hours**.

Check: **Access → Applications** lists `lab-n8n` with the `Owner` policy.

**3. Add the route to n8n.**
On the jotaeme-ingress tunnel, add a published application as its [operations guide](https://github.com/JuanMaVelezPa/jotaeme-ingress/blob/main/docs/operations.md#add-a-public-hostname) describes:

- **Subdomain:** `lab-n8n`, **Domain:** `jotaeme.dev`, empty path.
- **Service URL:** `http://n8n:5678`.

Check: **DNS** in the `jotaeme.dev` zone shows a proxied `CNAME lab-n8n` to `<tunnel id>.cfargotunnel.com`.
Opening `https://lab-n8n.jotaeme.dev` in a private window asks for an email, sends a PIN to `you@example.com`, and then shows the n8n sign-in page.

The n8n service already sets `N8N_HOST`, `N8N_PROTOCOL`, `N8N_EDITOR_BASE_URL`, `N8N_WEBHOOK_URL`, `N8N_PROXY_HOPS` and `N8N_SECURE_COOKIE` for this hostname in `docker-compose.yml`.
To use another hostname, change those values, the tunnel route and the Access application together.

## 5. Create the Google OAuth client

Both projects read a Google Sheet and send email with Gmail, through one OAuth client.
This part is done by hand, once. OAuth consent cannot be automated.

You need a Google account. It owns the sheets and sends the emails.

### Enable the Google APIs

1. Open https://console.cloud.google.com and create a project, for example `n8n`.
2. With that project selected, go to **APIs & Services → Library**.
3. Enable **Google Sheets API**.
4. Enable **Gmail API**.

### Create the OAuth client

1. Go to **APIs & Services → OAuth consent screen** (Google Auth Platform) and click **Get started**.
2. Fill in an app name (for example `n8n`) and your email as support email.
3. Choose **External** as audience, add your email as contact, and finish.
4. In **Audience**, add under **Test users** the Google account that owns the sheets and will send the emails.
5. In **Clients**, click **Create client**:
   - **Application type:** Web application.
   - **Authorized redirect URIs:** add both.
     - `https://lab-n8n.jotaeme.dev/rest/oauth2-credential/callback`
     - `http://localhost:5678/rest/oauth2-credential/callback`
6. Copy the **Client ID** and the **Client secret**.

While the app is in **Testing** status, Google expires the authorisation after 7 days and both credentials must be reconnected.
To avoid this, open **Audience** and click **Publish app**. An unverified app shows a warning screen on sign-in, which is fine for personal use.

## 6. Create the credentials in n8n

Both credentials use the same Client ID and Client secret.

1. In n8n, open **Credentials → Create credential**.
2. Choose **Google Sheets OAuth2 API**, name it `Google Sheets account`, paste the Client ID and Client secret, click **Sign in with Google** and accept the permissions.
3. Repeat with **Gmail OAuth2 API**, named `Gmail account`.

Sign in with the account you added as test user.
Open n8n at `https://lab-n8n.jotaeme.dev`, or on the server at `http://localhost:5678`.
n8n builds the redirect from `N8N_EDITOR_BASE_URL`, so the credential page shows `https://lab-n8n.jotaeme.dev/rest/oauth2-credential/callback` as **OAuth Redirect URL**.
Google must list that URI, or the sign-in fails with `redirect_uri_mismatch`.
The `localhost` URI stays authorized as a fallback, for when `N8N_EDITOR_BASE_URL` is removed while the tunnel is down.

Check: both credentials show **Account connected**.

Credentials are never stored in this repo. n8n keeps them encrypted in the `n8n_n8n_data` volume.

## 7. Next step

Set up each project with its own sheet and workflow:

- [News digest](../projects/news-digest/README.md#setup)
- [Daily gospel](../projects/daily-gospel/README.md#setup)
