# n8n projects

Self-hosted n8n and the automation projects that run on it.

## Start n8n

```bash
docker compose up -d
```

The editor is at http://localhost:5678. The container joins the external Docker network `ia`, shared with `ollama`.

## Requirements

- Docker with the Compose plugin.
- The external network, created once: `docker network create ia`.
- For projects that use AI, an `ollama` container on the same network with the model pulled:

  ```bash
  docker exec ollama ollama pull gemma4:12b
  ```

Ollama works on CPU. A GPU makes it faster, and needs the NVIDIA driver, the NVIDIA Container Toolkit and a GPU section in the `ollama` compose file.

## Layout

- `docker-compose.yml` — the n8n container, shared by all projects.
- `specs/` — one spec per feature, numbered across all projects.
- `projects/<name>/` — one folder per project, with its own `README.md`, `workflows/` and data templates.

Credentials are never stored here. n8n keeps them encrypted in the `n8n_data` volume.

## Projects

| Project | Spec | Description |
| --- | --- | --- |
| [`projects/news-digest/`](projects/news-digest/README.md) | [SPEC 01](specs/01-daily-news-digest.md) | Daily news email for each user in a Google Sheet, from RSS feeds, summarised with Ollama in the user's language and English level. |
| [`projects/daily-gospel/`](projects/daily-gospel/README.md) | [SPEC 02](specs/02-daily-gospel-email.md) | Daily email with the Catholic gospel of the day in English and Spanish, and a question-and-answer reflection written with Ollama for each group's language, level and audience. |
