# Setup

The shared setup for every project in this repo: Docker, n8n, Ollama and the Google credentials.
Do it once, in this order. Then follow the README of each project.

Commands are run from the repo root unless a step says otherwise.

## 1. Install Docker

Install Docker Engine and the Compose plugin by following https://docs.docker.com/engine/install/.

Check: both commands print a version.

```bash
docker --version
docker compose version
```

## 2. Create the Docker network

n8n and Ollama talk to each other over an external Docker network named `ia`.
Create it once:

```bash
docker network create ia
```

Check: `docker network ls` lists `ia`.

## 3. Start Ollama

Ollama runs in its own container, named `ollama`, on the `ia` network.
Its compose file is not part of this repo. Create it in a folder of your choice, for example `~/ollama/docker-compose.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    restart: unless-stopped
    ports:
      - "127.0.0.1:11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    networks:
      - ia

volumes:
  ollama_data:

networks:
  ia:
    external: true
```

Start it from that folder and pull the model used by the project templates:

```bash
docker compose up -d
docker exec ollama ollama pull gemma4:12b
```

Check: `docker exec ollama ollama list` shows `gemma4:12b`.

### GPU (optional)

Ollama works on CPU. A GPU makes each call several times faster.
For an NVIDIA GPU you need:

- The NVIDIA driver on the host.
- The NVIDIA Container Toolkit, from https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html.
- This block under the `ollama` service, then `docker compose up -d` again:

  ```yaml
      deploy:
        resources:
          reservations:
            devices:
              - driver: nvidia
                count: all
                capabilities: [gpu]
  ```

Check: while a workflow is running, `docker exec ollama ollama ps` shows `GPU` in the `PROCESSOR` column.

## 4. Start n8n

From the repo root:

```bash
docker compose up -d
```

Open http://localhost:5678 and create the owner account.

Check: n8n can reach Ollama over the `ia` network.

```bash
docker exec n8n wget -qO- http://ollama:11434/api/version
```

It prints the Ollama version as JSON.

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
   - **Authorized redirect URIs:** `http://localhost:5678/rest/oauth2-credential/callback`
6. Copy the **Client ID** and the **Client secret**.

While the app is in **Testing** status, Google expires the authorisation after 7 days and both credentials must be reconnected.
To avoid this, open **Audience** and click **Publish app**. An unverified app shows a warning screen on sign-in, which is fine for personal use.

## 6. Create the credentials in n8n

Both credentials use the same Client ID and Client secret.

1. In n8n, open **Credentials → Create credential**.
2. Choose **Google Sheets OAuth2 API**, name it `Google Sheets account`, paste the Client ID and Client secret, click **Sign in with Google** and accept the permissions.
3. Repeat with **Gmail OAuth2 API**, named `Gmail account`.

Sign in with the account you added as test user. Open n8n at `http://localhost:5678`, not at another host name or IP, or the redirect URI will not match.

Check: both credentials show **Account connected**.

Credentials are never stored in this repo. n8n keeps them encrypted in the `n8n_data` volume.

## 7. Next step

Set up each project with its own sheet and workflow:

- [News digest](../projects/news-digest/README.md#setup)
- [Daily gospel](../projects/daily-gospel/README.md#setup)
