# n8n projects

Self-hosted n8n and the automation projects that run on it.

## Start n8n

```bash
docker compose up -d
```

The editor is at http://localhost:5678. The container joins the external Docker network `ia`, shared with `ollama`.

## Layout

- `docker-compose.yml` — the n8n container, shared by all projects.
- `specs/` — one spec per feature, numbered across all projects.
- `projects/<name>/` — one folder per project, with its own `README.md`, `workflows/` and data templates.

Credentials are never stored here. n8n keeps them encrypted in the `n8n_data` volume.

## Projects

| Project | Spec | Description |
| --- | --- | --- |
| `projects/news-digest/` | SPEC 01 | Daily news email from RSS feeds, summarised with Ollama. |
