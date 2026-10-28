# SPEC 05 — GitHub documentation and repository metadata

> **Status:** Approved
> **Depends on:** SPEC 01, SPEC 02, SPEC 03, SPEC 04
> **Date:** 2026-10-27
> **Objective:** Rename the repository to `lab-n8n`, rewrite the root README, add shared guides in `docs/`, align both project READMEs with them, add an MIT license, and set the GitHub description, topics and homepage with the `gh` CLI, so a visitor can understand, install and run the repository from GitHub alone.

## Why this spec exists

The repository works, but its GitHub page has no description, no topics and no license.
The root README is a short note, and the shared setup lives inside one project.
The Google OAuth steps are written in the news digest README, and the daily gospel README points into the middle of it.
Operational facts that took time to learn, such as "import unpublishes the workflow and drops the sheet ID and credentials", are spread across project READMEs or not written down at all.

The user asked for this together with a domain and a Cloudflare Tunnel with authentication.
That second part is a different area (infrastructure and security), so it goes in SPEC 06.
This spec documents only what exists today.

## Scope

**In:**

- Rename the GitHub repository from `JuanMaVelezPa/n8n` to `JuanMaVelezPa/lab-n8n`, and point the local `origin` remote to the new URL.
- A rewrite of the root `README.md`.
- A new `docs/architecture.md`.
- A new `docs/setup.md`, with the shared setup: Docker network, n8n, Ollama and the Google OAuth credentials.
- A new `docs/operations.md`, with day-to-day operation of the stack.
- The Google OAuth steps (today's steps 2 to 4 of `projects/news-digest/README.md`) move to `docs/setup.md`.
- `projects/news-digest/README.md` and `projects/daily-gospel/README.md` keep only project-specific steps and link to `docs/setup.md` for the credentials.
- Both project READMEs get the same section order and the same heading names where the content is equivalent.
- A new `LICENSE` file with the MIT license.
- GitHub description and 12 topics, set with `gh repo edit`.
- Mermaid diagrams only. No images.
- Both workflow files in `projects/*/workflows/` refreshed from the version published in n8n, cleaned with the export filter of `docs/operations.md`, so the repo uses the real credential names `Google Sheets account` and `Gmail account`.

**Out of scope (for future specs):**

- The domain, Cloudflare Tunnel, Cloudflare Access and any authentication (SPEC 06).
- Any mention of remote access, the lab subdomains or a URL for this server. SPEC 06 adds them.
- Git version tags, GitHub Releases and a `CHANGELOG.md`.
- Screenshots of the emails or of the n8n editor.
- `CONTRIBUTING.md`, `SECURITY.md` and a `docs/spec-workflow.md` guide.
- A social preview image for the repository.
- GitHub Actions (link checking, Markdown lint).
- Renaming the local folder `/mnt/data/apps/n8n`.
- Any change to workflow logic, sheets templates, `docker-compose.yml` or specs 01 to 04.
- Translations of the documentation. All docs stay in English.

## Data model

This feature introduces no new runtime data. It adds and changes documentation files and repository settings.

**Files**

| Path | Change |
| --- | --- |
| `README.md` | Rewritten. |
| `LICENSE` | New. MIT, `Copyright (c) 2026 Juan Manuel Velez Parra`. |
| `docs/architecture.md` | New. |
| `docs/setup.md` | New. |
| `docs/operations.md` | New. |
| `projects/news-digest/README.md` | OAuth steps removed, links to `docs/setup.md`, sections aligned. |
| `projects/daily-gospel/README.md` | Credential section links to `docs/setup.md`, sections aligned. |
| `projects/news-digest/workflows/news-digest.json` | Refreshed from the published version: credential names only. |
| `projects/daily-gospel/workflows/daily-gospel.json` | Refreshed from the published version: credential names, sticky note positions and n8n defaults. |

**Root `README.md`**, in this order:

1. Title and one-line summary.
2. Static shields.io badges: `License: MIT`, `n8n`, `Ollama`, `Docker Compose`.
3. What it is: self-hosted n8n with local AI, and the two projects in two lines.
4. Architecture: one Mermaid `flowchart` with n8n, Ollama, Google Sheets, Gmail and the RSS / gospel sources, and a link to `docs/architecture.md`.
5. Quick start: the minimum commands to start n8n, and a link to `docs/setup.md` for the full path.
6. Requirements.
7. Repository layout, including `docs/`.
8. Projects table (as today, with links to the specs).
9. How the repo is built: one short paragraph about specs in `specs/`, numbered across projects, with their states.
10. Documentation index: links to the three `docs/` guides and both project READMEs.
11. License.

**`docs/architecture.md`** covers:

- Containers `n8n` and `ollama`, the external Docker network `ia`, and which ports each publishes.
- Where state lives: the `n8n_data` volume (workflows, encrypted credentials), the Google Sheets (users and configuration), and nothing in the repo.
- One Mermaid diagram per project with its node groups, from trigger to sent email.
- How the model is called: `POST /api/generate` on `http://ollama:11434`, model taken from the `Config` tab.

**`docs/setup.md`** covers, in order:

1. Docker and the Compose plugin.
2. `docker network create ia`.
3. The `ollama` container on the `ia` network and `ollama pull gemma4:12b`. GPU as optional, with what it needs.
4. `docker compose up -d` and opening `http://localhost:5678`.
5. Google Cloud project, Sheets and Gmail APIs, OAuth client and redirect URI (moved from the news digest README).
6. The two n8n credentials `Google Sheets account` and `Gmail account` (moved from the news digest README).
7. Next step: links to each project README.

**`docs/operations.md`** covers:

- Import a workflow from the repo, and what import does: it unpublishes the workflow and drops `sheetId` and the selected credentials.
- Export a workflow back to the repo after a change in the editor.
- Back up and restore the `n8n_data` volume into `backups/` (already in `.gitignore`).
- Update the n8n image with `docker compose pull` and `docker compose up -d`.
- Ollama notes: model choice, CPU versus GPU timing, and that its logs are in UTC while n8n runs in `America/Bogota`.
- Where to look when an email does not arrive: **Executions** in n8n, and the per-project "When something fails" sections.

**GitHub metadata**

```text
repository: JuanMaVelezPa/lab-n8n   (was JuanMaVelezPa/n8n)
description: Self-hosted n8n automations with local AI (Ollama): a daily news digest and a bilingual daily gospel email, driven by Google Sheets and built spec-first.
topics: n8n, n8n-workflows, ollama, local-llm, self-hosted, docker-compose, automation, google-sheets, gmail, rss, email-newsletter, spec-driven-development
homepage: https://jotaeme.dev
```

Conventions:

- Every new section uses one sentence per line, as the existing READMEs do.
- Links between docs are relative, so they work on GitHub and in a local clone.
- Real email addresses, sheet IDs, OAuth client IDs and secrets never appear. Examples use `you@example.com` and `<document ID>`.
- Commands are written to be run from the repo root.

## Implementation plan

1. Rename the repository on GitHub to `lab-n8n` (**Settings → General → Repository name**, or `gh repo rename lab-n8n` once `gh` is logged in). Run `git remote set-url origin https://github.com/JuanMaVelezPa/lab-n8n.git`. Check: `git remote -v` shows the new URL and `git fetch` works.
2. Add `LICENSE` with the MIT text. Check: the file starts with `MIT License` and has the copyright line from the Data model.
3. Create `docs/setup.md` with the shared setup. Move the Google APIs, OAuth client and n8n credential steps out of `projects/news-digest/README.md` into it. In the news digest README, replace them with one short step that links to `docs/setup.md`. Renumber the remaining steps. Check: no text about the OAuth consent screen is left in the news digest README, and every link in both files resolves.
4. In `projects/daily-gospel/README.md`, change the credentials link to `docs/setup.md`. Align the section order and heading names of both project READMEs. Check: both READMEs have `How it works`, `Setup`, `Daily use` and `When something fails` as top-level sections, in that order.
5. Create `docs/operations.md`. Move the generic import, re-publish and export text out of the project READMEs where it is the same for both. Each project README keeps only its own file names and node names. Check: the import commands in both READMEs still work when copied into a terminal.
6. Create `docs/architecture.md` with the stack diagram and one diagram per project. Check: the diagrams render in the GitHub preview of the branch.
7. Rewrite the root `README.md` with the sections in the Data model. Check: the README renders on GitHub, the badges load and every link resolves.
8. The user runs `! gh auth login`. Then run `gh repo edit JuanMaVelezPa/lab-n8n` with `--description`, `--homepage` and `--add-topic` for each topic in the Data model. Check: `gh repo view JuanMaVelezPa/lab-n8n --json description,homepageUrl,repositoryTopics,licenseInfo` shows the description, the homepage, the 12 topics and `MIT` (after the branch is merged to `main`).

## Acceptance criteria

- [ ] `https://github.com/JuanMaVelezPa/lab-n8n` opens the repository, and `https://github.com/JuanMaVelezPa/n8n` redirects to it.
- [ ] `git remote get-url origin` returns `https://github.com/JuanMaVelezPa/lab-n8n.git`.
- [ ] `LICENSE` exists at the repo root and GitHub shows `MIT license` in the repository sidebar.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json description` returns the exact description in the Data model.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json repositoryTopics` returns exactly the 12 topics in the Data model.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json homepageUrl` returns `https://jotaeme.dev`.
- [ ] `docs/architecture.md`, `docs/setup.md` and `docs/operations.md` exist.
- [ ] The root `README.md` has the 11 sections of the Data model, in that order.
- [ ] Every Mermaid diagram renders on the GitHub page, with no "Unable to render" error.
- [ ] Every relative link in `README.md`, `docs/*.md` and `projects/*/README.md` points to a file that exists, and every `#anchor` matches a heading in that file.
- [ ] The text `OAuth consent screen` appears in `docs/setup.md` and in no project README.
- [ ] Both project READMEs link to `docs/setup.md` for the Google credentials.
- [ ] Both project READMEs have the top-level sections `How it works`, `Setup`, `Daily use` and `When something fails`, in that order.
- [ ] `docs/operations.md` says that importing a workflow unpublishes it and drops `sheetId` and the selected credentials.
- [ ] `docs/operations.md` has commands to back up and restore the `n8n_data` volume into `backups/`.
- [ ] `git grep -nE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[a-z]{2,}' -- '*.md'` finds only `example.com` and `anthropic.com` addresses.
- [ ] `README.md`, `docs/*.md` and `projects/*/README.md` do not contain the words `tunnel`, `Cloudflare` or `lab-n8n.jotaeme.dev`.
- [ ] No file under `projects/*/sheets/`, `specs/01-*` to `specs/04-*` or `docker-compose.yml` is changed.
- [ ] The files in `projects/*/workflows/` differ from `main` only in credential names, node positions and settings n8n adds on export. They still hold the `<Google Sheet document ID>` placeholder and no credential IDs.
- [ ] Docs, project READMEs and workflow files name the credentials `Google Sheets account` and `Gmail account`, as in n8n.
- [ ] The repo contains no image files added by this spec.

## Decisions

- **Yes:** split the request into two specs, docs first. Chosen by the user. Docs and remote access are different areas with different risks.
- **No:** one combined spec. Two unrelated areas are harder to verify and review together.
- **Yes:** the domain and tunnel go in SPEC 06. The user already owns the domain and it is managed in Cloudflare.
- **Yes:** keep one repository for all projects. They share `docker-compose.yml`, the `ia` network, the Google credentials and the spec numbering, and deployment is the same `n8n import:workflow` for one repo or many.
- **No:** one repository per project. Shared setup would be copied or need a third repository, with no deploy benefit, because self-hosted n8n (without Enterprise source control) does not pull from Git.
- **Yes:** rename the repository to `lab-n8n`. Chosen by the user. `n8n` alone looks like a fork of the official n8n repository. `lab-n8n` matches the "Lab" card on jotaeme.dev and the `lab-<name>.jotaeme.dev` naming planned for SPEC 06, where n8n will be `lab-n8n.jotaeme.dev`.
- **No:** `n8n-ollama-automations`. Not every future project will use Ollama.
- **No:** `n8n-homelab`, `n8n-lab`, `n8n-automations` and `selfhosted-n8n`. Considered; `lab-n8n` keeps the same pattern as the subdomain.
- **Yes:** homepage set to `https://jotaeme.dev`. Chosen by the user. The lab will be private, so the public site is the useful link for visitors.
- **No:** homepage left empty until SPEC 06, or pointing to the lab. A private URL only shows a login to visitors.
- **Yes:** MIT license. Chosen by the user. Permissive and common for personal automation repos.
- **No:** Apache-2.0 or no license. Not selected.
- **Yes:** "tags" means GitHub topics. Chosen by the user.
- **No:** git version tags, GitHub Releases and a changelog. Not selected; they can come in their own spec.
- **Yes:** the description and topics in the Data model. Chosen by the user.
- **Yes:** metadata applied with `gh repo edit` after `gh auth login`. Chosen by the user. The spec records the exact values, so the change can be repeated.
- **No:** editing the metadata in the GitHub web UI. Not reproducible from the repo.
- **Yes:** root README rewrite, `docs/` guides and per-project README polish. Chosen by the user.
- **No:** `CONTRIBUTING.md`, `SECURITY.md` and `docs/spec-workflow.md`. Not selected. The root README has one paragraph on specs instead.
- **Yes:** `docs/architecture.md`, `docs/setup.md` and `docs/operations.md`. Chosen by the user.
- **Yes:** the Google OAuth steps move to `docs/setup.md`. Chosen by the user. Both projects use the same credentials, so they belong to the shared setup.
- **No:** keeping the OAuth steps in the news digest README. The daily gospel README would keep pointing into another project.
- **Yes:** no images, only Mermaid diagrams. Chosen by the user. Mermaid renders on GitHub and cannot leak a real address.
- **No:** screenshots. Risk of showing personal data, and they go out of date.
- **Yes:** no mention of remote access before SPEC 06. Chosen by the user. The docs describe only what exists.
- **Yes:** static shields.io badges for license, n8n, Ollama and Docker Compose. They load from a URL and add no file to the repo.
- **No:** dynamic badges such as CI status. There is no CI.
- **Yes:** use the credential names that exist in n8n, `Google Sheets account` and `Gmail account`, in docs and workflow files. Chosen by the user during implementation. The docs describe the real install.
- **No:** renaming the live credentials to `Google Sheets OAuth2` and `Gmail OAuth2`. Not selected.
- **Yes:** refresh both workflow files from the published version in this spec. Chosen by the user during implementation. The repo had drifted from n8n (credential names, Daily Gospel note positions).
- **Yes:** documentation in English. The existing READMEs and specs are in English.

## Risks

| Risk | Mitigation |
| --- | --- |
| Moving the OAuth steps breaks links from other places, such as old issues or the daily gospel README. | Step 3 and the link criterion check every relative link and anchor. |
| Old links and clones use `JuanMaVelezPa/n8n`. | GitHub redirects the old name as long as no new repository takes it. The remote is updated in step 1. |
| `gh` is not logged in, so step 8 cannot run. | The user runs `! gh auth login` first. Steps 2 to 7 do not need it, and step 1 can be done in the web UI. |
| GitHub only detects the license after `LICENSE` is on `main`. | The license criterion is checked after the merge. |
| A Mermaid diagram that renders locally fails on GitHub. | Step 6 checks the diagrams in the GitHub preview of the branch. |
| A real address or ID is copied into an example. | The email grep criterion, and the convention to use `you@example.com` and `<document ID>`. |
| The OAuth redirect URI in `docs/setup.md` (`http://localhost:5678/...`) changes when SPEC 06 adds a domain. | SPEC 06 updates `docs/setup.md`. This spec documents today's value. |

## What is **not** in this spec

- Domain, Cloudflare Tunnel, Cloudflare Access or any authentication.
- Git tags, Releases or a changelog.
- Screenshots or a social preview image.
- `CONTRIBUTING.md`, `SECURITY.md` or a spec workflow guide.
- GitHub Actions.
- Changes to workflow logic, sheet templates, `docker-compose.yml` or earlier specs.
- Renaming the local folder.
- Translations.

Each one of those, if it lands, goes in its own spec.
