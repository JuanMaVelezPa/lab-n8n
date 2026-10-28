# SPEC 07 — Public repository

> **Status:** Approved
> **Depends on:** SPEC 05, SPEC 06
> **Date:** 2026-10-28
> **Objective:** Make `JuanMaVelezPa/lab-n8n` public after a clean gitleaks scan of its full history, with secret scanning, push protection and a protected `main` turned on.

## Why this spec exists

The repository is private today.
SPEC 05 wrote the docs, license and metadata for a GitHub visitor, and SPEC 06 published n8n behind Cloudflare Access, so the repo is ready to be read by others.
Making it public cannot be undone in practice: forks, clones and caches keep what was visible.
So this spec checks the history first and fixes the settings that only matter once strangers can read and push requests to the repo.

A manual review before writing this spec found:

- No tunnel token, OAuth client ID, credential ID or sheet ID in any commit. Workflow files hold only the `<Google Sheet document ID>` placeholder.
- The personal Gmail address appears only as the author and committer email of commits, not in any file.
- `lab-n8n.jotaeme.dev` appears in `README.md`, `docs/*.md` and `docker-compose.yml`.
- Six merged `spec-*` branches and six pull requests exist on GitHub.

## Scope

**In:**

- A gitleaks scan of every commit on every branch, run once in Docker, before the visibility changes.
- The repository visibility changed to public with `gh repo edit`.
- Secret scanning and push protection turned on.
- A repository ruleset on the default branch that blocks deletion and force-push.
- The Projects tab turned off.

**Out of scope (for future specs):**

- Hiding the Gmail address: no history rewrite, no noreply address in the git config, no change to the GitHub account email settings.
- Replacing `lab-n8n.jotaeme.dev` with a placeholder or moving it to `.env`.
- Deleting the merged `spec-*` branches or turning on automatic deletion of head branches.
- `SECURITY.md`, `CONTRIBUTING.md`, issue templates or a code of conduct.
- Dependabot alerts or updates.
- Required pull request reviews or required status checks on `main`.
- GitHub Actions, including a gitleaks workflow on every push.
- Any change to files in the repo other than this spec.

## Data model

This feature introduces no new runtime data and no new files in the repo besides this spec.
It changes GitHub settings only.

**Repository settings** (`JuanMaVelezPa/lab-n8n`)

```text
visibility:                         public            (was private)
has_projects:                       false             (was true)
has_issues:                         true              (unchanged)
security_and_analysis:
  secret_scanning:                  enabled
  secret_scanning_push_protection:  enabled
```

**Repository ruleset** (target shape for `POST /repos/JuanMaVelezPa/lab-n8n/rulesets`)

```json
{
  "name": "Protect main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [{ "type": "deletion" }, { "type": "non_fast_forward" }]
}
```

No bypass actors, so the rule also applies to the owner.
Pull requests are not required, so direct pushes and the usual `gh pr merge` keep working.

**gitleaks scan**

```text
image:   ghcr.io/gitleaks/gitleaks:latest
command: git /repo --redact --verbose --report-path /repo-report/gitleaks.json
mounts:  repo root → /repo (read-only), scratchpad folder → /repo-report
scope:   every commit reachable from every local and remote-tracking branch
```

The report is written to the scratchpad, never to the repo.

Conventions:

- GitHub settings are changed with `gh` from the server, so the spec records the exact values and the change can be repeated.
- The visibility changes only after the gitleaks scan exits with code `0`.

## Implementation plan

1. Run `git fetch --all` so every remote branch is present locally. Run the gitleaks scan from the Data model. Check: the command exits with code `0` and the report lists no findings. If it finds a leak, stop: the repo stays private, the secret is rotated, and the fix goes in its own spec.
2. Commit this spec's status change on the branch `spec-07-public-repository` and push it. Check: the push is accepted.
3. Run `gh repo edit JuanMaVelezPa/lab-n8n --visibility public --accept-visibility-change-consequences`. Check: `gh repo view JuanMaVelezPa/lab-n8n --json visibility` returns `PUBLIC`.
4. Turn on secret scanning and push protection with `gh api -X PATCH repos/JuanMaVelezPa/lab-n8n` and the `security_and_analysis` values from the Data model. Check: `gh api repos/JuanMaVelezPa/lab-n8n --jq .security_and_analysis` shows both as `enabled`.
5. Create the ruleset from the Data model with `gh api -X POST repos/JuanMaVelezPa/lab-n8n/rulesets`. Check: `gh api repos/JuanMaVelezPa/lab-n8n/rules/branches/main --jq '.[].type'` prints `deletion` and `non_fast_forward`.
6. Run `gh repo edit JuanMaVelezPa/lab-n8n --enable-projects=false`. Check: `gh repo view JuanMaVelezPa/lab-n8n --json hasProjectsEnabled` returns `false`.
7. Merge the `spec-07-public-repository` pull request into `main`. Check: the merge is accepted with the ruleset active.

## Acceptance criteria

- [ ] The gitleaks scan of every branch exited with code `0` before the visibility changed.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json visibility` returns `PUBLIC`.
- [ ] `curl -s -o /dev/null -w '%{http_code}' https://github.com/JuanMaVelezPa/lab-n8n` without a GitHub login returns `200`.
- [ ] `gh api repos/JuanMaVelezPa/lab-n8n --jq .security_and_analysis.secret_scanning.status` returns `enabled`.
- [ ] `gh api repos/JuanMaVelezPa/lab-n8n --jq .security_and_analysis.secret_scanning_push_protection.status` returns `enabled`.
- [ ] `gh api repos/JuanMaVelezPa/lab-n8n/rules/branches/main --jq '.[].type'` lists `deletion` and `non_fast_forward`.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json hasProjectsEnabled` returns `false`.
- [ ] `gh repo view JuanMaVelezPa/lab-n8n --json hasIssuesEnabled` returns `true`.
- [ ] The six `spec-01-*` to `spec-06-*` branches still exist on GitHub.
- [ ] `git diff main~1 --stat` after the merge shows only `specs/07-public-repository.md`.
- [ ] The scheduled runs of both workflows after the change send their emails, with no failed execution in **Executions**.

## Decisions

- **Yes:** the Gmail address stays visible as the commit email, in the history and in future commits. Chosen by the user. It does not need to be private.
- **No:** rewrite history with `git filter-repo`. It changes every hash, needs a force-push, and GitHub keeps `refs/pull/N/head` for PRs #1 to #6 anyway.
- **No:** the GitHub noreply address for future commits and the account email privacy settings. Not needed, because the address does not need to be private.
- **No:** a new repository with one squashed commit. It loses the commit and PR history that the specs link to.
- **Yes:** keep `lab-n8n.jotaeme.dev` in the docs and `docker-compose.yml`. Chosen by the user. It is behind Cloudflare Access with a PIN for one email, the hostname is already in public Certificate Transparency logs, and the docs stay concrete.
- **No:** move the hostname to `.env` and use a placeholder in the docs. It touches compose, `.env.example` and every guide for little gain.
- **Yes:** a gitleaks scan of the full history before the change. Chosen by the user. The manual review is not a tool and can miss token formats.
- **Yes:** run gitleaks once in Docker. No new tool installed on the server and nothing added to the repo.
- **No:** a gitleaks GitHub Action. Push protection covers new pushes, and Actions are out of scope since SPEC 05.
- **Yes:** secret scanning with push protection. Chosen by the user. Free on public repositories, and it blocks a pushed token such as `TUNNEL_TOKEN`.
- **Yes:** protect `main` against deletion and force-push with a ruleset. Chosen by the user. It does not require reviews, so the one-person PR flow keeps working.
- **No:** classic branch protection. Rulesets are GitHub's current mechanism and are created with one API call.
- **No:** required reviews or status checks. There is one maintainer and no CI.
- **Yes:** turn off the Projects tab. Chosen by the user. The repo does not use it.
- **Yes:** keep Issues on. Visitors can report problems.
- **Yes:** keep the merged `spec-*` branches. Chosen by the user.
- **No:** delete the merged branches and turn on automatic head branch deletion. Not selected.
- **No:** `SECURITY.md` and Dependabot alerts. Not selected. The only dependencies are Docker images.
- **Yes:** the agent changes the visibility with `gh`. Chosen by the user. The step is recorded and runs only after a clean scan.
- **Yes:** secret scanning, the ruleset and the Projects change come after the visibility change. Some of these features need a paid plan on private repositories of a personal account.

## Risks

| Risk | Mitigation |
| --- | --- |
| gitleaks finds a real secret. | Step 1 stops the spec. The repo stays private, the secret is rotated, and the cleanup goes in its own spec. |
| gitleaks reports a false positive, such as an example value in a doc. | Check the finding by hand. If it is not a secret, record it in the PR description and go on. No `.gitleaksignore` file is added. |
| The public Gmail address in commits gets more spam. | Accepted by the user. |
| Publishing makes `lab-n8n.jotaeme.dev` easy to find, so bots try it. | Cloudflare Access stops every request before n8n. n8n keeps its own login behind it. |
| A forced push is needed one day to fix `main`. | Turn off the ruleset in **Settings → Rules**, push, and turn it on again. |
| A public repo can be forked; forks keep any secret pushed later. | Push protection blocks known token formats before they land. `.env` stays ignored. |
| The visibility change resets stars, watchers or other settings. | The repo has none. `--accept-visibility-change-consequences` is required by `gh` for that reason. |

## What is **not** in this spec

- Any change to how the Gmail address appears in commits.
- Placeholder or `.env` hostname.
- Deleting merged branches.
- `SECURITY.md`, `CONTRIBUTING.md`, issue templates or a code of conduct.
- Dependabot.
- Required reviews or status checks.
- GitHub Actions.
- Changes to any repo file other than this spec.

Each one of those, if it lands, goes in its own spec.
