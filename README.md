# Cliq Connector Template

This repository contains a ready-to-use GitHub Actions workflow for:

- posting GitHub events to Zoho Cliq,
- running AI PR review checks,
- storing/reusing Cliq PR thread IDs with GitHub Project V2.

The canonical workflow is [TemplateRepositoryNew/.github/workflows/CliqConnector.yml](TemplateRepositoryNew/.github/workflows/CliqConnector.yml).

## 1. Create GitHub Token (PROJECT_TOKEN)

Use a Personal Access Token (classic) from the account that owns or can edit the Project V2.

1. Open <https://github.com/settings/tokens>
2. Create a classic PAT.
3. Enable scopes:
   - `repo` (required)
   - `project` (required)
   - `read:org` (only if your project owner is an organization)

Why this token is needed:

- Project V2 GraphQL read/write (find project, add PR item, update field value).

## 2. Add Secrets via a Protected Environment (Admin-Only)

`ENDPOINT`, `PROJECT_TOKEN`, and `AI_REVIEW_TOKEN` are sensitive (webhook/bot
auth, GitHub PAT, AI provider key). Store them as **Environment secrets**
inside a protected GitHub Environment instead of plain repository secrets, so
only repo/org Admins (or people you explicitly add as required reviewers) can
view the secret list or change values.

Note: this is different from GitHub Actions **Variables** (`vars.*`).
Variables are plain text, visible to anyone with read access to the repo, and
are never masked in logs. Never put a token in `vars.*`. Both repository
secrets and environment secrets are masked identically in logs (GitHub
replaces the exact value with `***` wherever it appears in step output) — the
difference is access control, not masking:

| Storage | Masked in logs | Who can view the value | Who can create/edit |
|---|---|---|---|
| Repository secret | Yes | Nobody (write-only) | Anyone with repo Write access |
| **Environment secret** | Yes | Nobody (write-only) | Only Admins / configured required reviewers |
| Variable (`vars.*`) | No | Anyone with read access | Anyone with repo Write access |

### 2.1 Create the environment

1. Go to `Settings` -> `Environments` -> `New environment`.
2. Name it `cliq-production` (must match `environment:` in
   [CliqConnector.yml](.github/workflows/CliqConnector.yml); rename both together
   if you use a different name).
3. Under **Deployment protection rules**:
   - Add **Required reviewers** and list only Admins/maintainers who should be
     able to approve workflow runs that use these secrets.
   - Optionally restrict **Deployment branches and tags** to `main` (or your
     protected branch) so PRs from forks/other branches cannot access the
     environment's secrets.
4. Confirm repository settings so only users with the **Admin** role can
   create/edit Environments (`Settings` -> `Actions` -> `General`, and
   branch/role permissions as needed) — this is what prevents non-admin
   collaborators from modifying the secret values, since Environment secrets
   can only be managed from within the Environment's own settings page.

### 2.2 Add the environment secrets

Inside `Settings` -> `Environments` -> `cliq-production` -> `Environment secrets`,
add:

- `ENDPOINT`
  - Value format: `<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>`
- `PROJECT_TOKEN`
  - Value: classic PAT from Step 1
- `AI_REVIEW_TOKEN`
  - Value: provider token (OpenAI/Claude/Gemini), only if AI review is enabled

No workflow syntax changes are needed beyond the job-level `environment:` key
already present in `CliqConnector.yml` — `${{ secrets.ENDPOINT }}`,
`${{ secrets.PROJECT_TOKEN }}`, and `${{ secrets.AI_REVIEW_TOKEN }}` continue
to resolve exactly as before, just sourced from the environment instead of the
repository.

If a required reviewer gate is enabled, runs against `pull_request_target`
will pause with **"Waiting for approval"** until an admin approves — this is
expected and is the access-control boundary your org wanted.

Cliq auth modes:

1. Webhook mode
   - `ENDPOINT=<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>`
   - `CLIQ_AUTH_TOKEN` not required

2. Bot-auth mode
   - user must create the bot in Cliq
   - user must add the bot to the target channel
  - `ENDPOINT=<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>`
  - workflow appends `bot_unique_name` query parameter from `CLIQ_BOT_UNIQUE_NAME`

In bot-auth mode, the message is posted using bot authentication. In webhook mode, the message can still display a bot block, but authentication is via webhook.

## 2B. Available Regions

Zoho Cliq is available in multiple regions. Use the same base domain as your Cliq organization URL when building the `ENDPOINT` secret.

Common region domains:

- US: `https://cliq.zoho.com`
- IN: `https://cliq.zoho.in`
- EU: `https://cliq.zoho.eu`
- AU: `https://cliq.zoho.com.au`
- JP: `https://cliq.zoho.jp`

Webhook endpoint format:

- `<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>`

Examples:

- US: `https://cliq.zoho.com/api/v2/channelsbyname/engineering/message?zapikey=1001.xxxxx`
- IN: `https://cliq.zoho.in/api/v2/channelsbyname/engineering/message?zapikey=1001.xxxxx`

Bot mode note:

- In bot mode, workflow appends `bot_unique_name=<BOT_UNIQUE_NAME>` automatically.
- Do not hardcode a different region in workflow; keep `ENDPOINT` in the same region as your Cliq org.

## 2A. What Changed Now (v3)

The workflow supports both user mode and bot mode. It currently defaults to user mode unless `CLIQ_NOTIFICATION_MODE` is set explicitly.

See [Section 2A-1](#2a-1-configuration-variables-reference) below for a full reference table of all four configuration variables (`CLIQ_NOTIFICATION_MODE`, `CLIQ_BOT_UNIQUE_NAME`, `CLIQ_THREAD_FIELD_ID`, `CLIQ_PROJECT_NUMBER`), where to set them, and their defaults.

Where changed:

- [TemplateRepositoryNew/.github/workflows/CliqConnector.yml](TemplateRepositoryNew/.github/workflows/CliqConnector.yml)

What was added:

1. `CLIQ_NOTIFICATION_MODE` (default: `user`) at job env level.
2. `CLIQ_BOT_UNIQUE_NAME` from repository/organization variable.
3. Endpoint preparation step that appends `bot_unique_name=<CLIQ_BOT_UNIQUE_NAME>` to `ENDPOINT`.
4. Validation that fails fast if bot mode is selected and bot unique name is missing.
5. Redacted endpoint debug preview in logs to verify runtime resolution.

What you must configure:

1. Secret `ENDPOINT` with your channel endpoint containing `zapikey`.
2. Variable `CLIQ_BOT_UNIQUE_NAME` with your real Cliq bot unique name.
3. Optional variable `CLIQ_NOTIFICATION_MODE` (`bot` or `user`).

If you set `CLIQ_NOTIFICATION_MODE=bot` and do not set `CLIQ_BOT_UNIQUE_NAME`, the workflow intentionally fails. If you keep user mode, bot unique name is not required.

## 2A-1. Configuration Variables Reference

All 4 values below are **GitHub Actions Variables** (`vars.*`), configured
**once** by the person setting up this workflow in their own repository
(usually a repo/org Admin or maintainer) — not by individual contributors,
and not by editing `CliqConnector.yml` directly. You never need to touch the
YAML to set these; the workflow already references them via `vars.NAME`.

Set them at: `Settings` -> `Secrets and variables` -> `Actions` -> **Variables** tab
(as distinct from the **Secrets** tab used in Section 2 — see the table there
for why tokens must never go here).

| Variable name | Used as (in YAML) | Required? | Default if unset | Purpose |
|---|---|---|---|---|
| `CLIQ_NOTIFICATION_MODE` | `CLIQ_NOTIFICATION_MODE` | No | `'user'` | Selects `user` (webhook auth) or `bot` (bot auth) posting mode. See Cliq auth modes in Section 2. |
| `CLIQ_BOT_UNIQUE_NAME` | `CLIQ_BOT_UNIQUE_NAME` | Only if `CLIQ_NOTIFICATION_MODE=bot` | `''` (empty) | Your Cliq bot's unique name; appended as `bot_unique_name=` query param on the endpoint. Workflow fails fast if mode is `bot` and this is blank. |
| `CLIQ_THREAD_FIELD_ID` | `GITHUB_PROJECT_THREAD_FIELD_ID` | **Yes — no default** | none | Numeric identifier from the Project field settings URL, or a GraphQL node id (`PVTF_*`), identifying the "Cliq Thread ID" custom field on your GitHub Project V2. Must be created by you before the workflow can update thread references correctly. |
| `CLIQ_PROJECT_NUMBER` | `GITHUB_PROJECT_NUMBER` | **Yes — no default** | none | Your GitHub Project V2 number (the number in the project's URL, e.g. the `1` in `https://github.com/orgs/OrgZylker/projects/1`). Must be created by you; the workflow will not guess a project. |

Notes:

- `CLIQ_NOTIFICATION_MODE` and `CLIQ_BOT_UNIQUE_NAME` use the
  `${{ vars.X || 'default' }}` pattern, so a missing variable falls back to
  the default shown above. `CLIQ_THREAD_FIELD_ID` and `CLIQ_PROJECT_NUMBER`
  have **no fallback** — they are read as `${{ vars.X }}` with nothing after
  the `||`, so you must create both of these two variables yourself before
  first use. Validation for other required-in-context values (like
  `CLIQ_BOT_UNIQUE_NAME` in bot mode) happens at runtime in the
  **Prepare Cliq endpoint** step, with a clear failure message.
- These are plain-text **Variables**, not **Secrets** — anyone with read
  access to the repo can see their values, and they appear unmasked in
  workflow run logs. This is intentional: none of these four values are
  credentials. Do not put a token or webhook key in a `vars.*` entry — see
  Section 2 for where the actual sensitive values (`ENDPOINT`,
  `PROJECT_TOKEN`, `AI_REVIEW_TOKEN`) belong.
- Because these are plain repository/organization Variables (not
  environment-scoped), they are **not** gated by the `environment:
  cliq-production` protection described in Section 2 — any repo collaborator
  with Write access can create or edit them. That's acceptable here since
  none of the four are sensitive; if you want stricter control anyway,
  GitHub also supports environment-level Variables, referenced the same way
  once the job declares that environment.
- Setting up each variable is a one-time action per repository — you do not
  need to repeat it per pull request, per branch, or per workflow run.

## 3. Configure Workflow Project Settings

For a quick summary of all configurable variables (including the two covered
in detail below), see [Section 2A-1](#2a-1-configuration-variables-reference).

In [TemplateRepositoryNew/.github/workflows/CliqConnector.yml](TemplateRepositoryNew/.github/workflows/CliqConnector.yml), keep:

- `CLIQ_THREAD_STORAGE_MODE: project`
- `GITHUB_PROJECT_OWNER: ${{ github.repository_owner }}`

Project field identifier source:

- **Do not edit `CliqConnector.yml` to set this value.** The YAML already
  contains a fixed reference — `GITHUB_PROJECT_THREAD_FIELD_ID: ${{
  vars.CLIQ_THREAD_FIELD_ID }}` — and that line should stay exactly as-is.
  It is only a *pointer* to a GitHub Actions Variable; it does not carry the
  actual field ID itself.
- `vars.CLIQ_THREAD_FIELD_ID` means a GitHub Actions Variable named `CLIQ_THREAD_FIELD_ID`.
  The **only** place its actual value is supplied is the repo/org
  `Settings` -> `Secrets and variables` -> `Actions` -> **Variables** tab
  (see steps below) — never inside the workflow file.
- **This variable is required — there is no default value.** You must create
  it before running the workflow. Set it to either:
  - the numeric field identifier from the Project field settings URL, or
  - the GraphQL node id directly (`PVTF_*`).

Example:

- Project field settings URL:
  - `https://github.com/users/<owner>/projects/<number>/settings/fields/377241900`
- Here `377241900` can be used as `CLIQ_THREAD_FIELD_ID`.

When this identifier is provided, the action resolves the correct GraphQL field node id internally and uses it to update the project item.

### One-time setup: create the `CLIQ_THREAD_FIELD_ID` Variable (not a yml edit)

This is a **Settings-only** action, done once per repository. You are **not**
editing `CliqConnector.yml` at any point in this process — the YAML keeps its
existing `${{ vars.CLIQ_THREAD_FIELD_ID }}` reference untouched, forever. You
are only supplying the value that reference reads at runtime, and that value
lives exclusively in the repo/org Variables store, never in the workflow file.

1. Go to repository (or organization) **Settings** (the GitHub web UI page —
   not `CliqConnector.yml`, not any file in this repo).
2. Open `Secrets and variables` -> `Actions` -> the **Variables** tab.
3. Click **New repository variable** (or **New organization variable**):
   - Name: `CLIQ_THREAD_FIELD_ID`
   - Value: the numeric identifier from the field settings URL (see example above)
4. Save. That's it — no commit, no PR, no yml change required.

Because the workflow already reads `${{ vars.CLIQ_THREAD_FIELD_ID }}`, the
same YAML works unmodified for every repository that adopts this template;
only the Variable's value differs per repository.

Project number source:

- **Do not edit `CliqConnector.yml` to set this value.** The YAML already
  contains a fixed reference — `GITHUB_PROJECT_NUMBER: ${{
  vars.CLIQ_PROJECT_NUMBER }}` — and that line should stay exactly as-is.
  It is only a *pointer* to a GitHub Actions Variable; it does not carry the
  actual project number itself.
- `vars.CLIQ_PROJECT_NUMBER` means a GitHub Actions Variable named `CLIQ_PROJECT_NUMBER`.
  The **only** place its actual value is supplied is the repo/org
  `Settings` -> `Secrets and variables` -> `Actions` -> **Variables** tab
  (see steps below) — never inside the workflow file.
- **This variable is required — there is no default value.** You must create
  it with your actual GitHub Project V2 number before running the workflow.
  Find the number in your project's URL, e.g. the `1` in
  `https://github.com/orgs/<orgname>/projects/1` (or
  `https://github.com/users/<username>/projects/1` for a user-owned project).

Example:

- Project URL: `https://github.com/orgs/OrgZylker/projects/1`
- Here `1` (the number right after `/projects/`) is the value to use as
  `CLIQ_PROJECT_NUMBER`.

### One-time setup: create the `CLIQ_PROJECT_NUMBER` Variable (not a yml edit)

Same rule as above: this is a **Settings-only** action, done once per
repository, and it never touches `CliqConnector.yml`.

1. Go to repository (or organization) **Settings** (the GitHub web UI page —
   not `CliqConnector.yml`, not any file in this repo).
2. Open `Secrets and variables` -> `Actions` -> the **Variables** tab.
3. Click **New repository variable** (or **New organization variable**):
   - Name: `CLIQ_PROJECT_NUMBER`
   - Value: your project number — e.g. for
     `https://github.com/orgs/OrgZylker/projects/1`, the value is `1`
4. Save. That's it — no commit, no PR, no yml change required.

Because the workflow already reads `${{ vars.CLIQ_PROJECT_NUMBER }}`, the same
YAML works unmodified for every repository that adopts this template; only
the Variable's value differs per repository.

The workflow uses auto-resolve mode for Project ID and Field ID via GraphQL.

## 4. Required Workflow Permissions

The workflow must include:

```yaml
permissions:
  contents: read
  checks: write
  issues: write
  pull-requests: write
  repository-projects: write
```

## 5. Event Flow

Current recommended PR event source is `pull_request_target` only (to avoid duplicate opened/closed notifications).

Flow on PR events:

1. Ensure PR is present in Project table.
2. Post notification to Cliq.
3. Resolve existing thread ID from Project field.
4. If unavailable, fallback to PR marker comment storage.
5. Save/update thread ID (project first, marker fallback).
6. Run AI review gate and publish check run.

## 6. Verification Checklist

After opening a PR:

1. Action logs show `EventNameRaw=pull_request_target`.
2. `Ensure PR is in Project table` step shows:
   - `PR inserted into project table successfully.` or
   - `PR already exists in project table.`
3. Cliq integration logs show:
   - `Saved Cliq thread id in GitHub Project custom field.` (preferred), or
   - fallback marker save message when project field write fails.
4. On `synchronize` and `closed`, messages should continue in the same thread.

## 7. Troubleshooting

If PR is not added to project table:

- Verify `PROJECT_TOKEN` is classic PAT with `repo` + `project` scopes.
- Verify project owner is correct and `CLIQ_PROJECT_NUMBER` variable matches your actual project number.
- Check GraphQL debug step output for NOT_FOUND / permission errors.

If thread ID is not reused:

- Confirm `CLIQ_THREAD_FIELD_ID` matches the numeric identifier from the Project field settings URL, or provide the GraphQL node id directly.
- Check for `Saved Cliq thread id in GitHub Project custom field.` in logs.
- If absent, check project field resolution logs.

If merge shows AI check as expected/waiting:

- Ensure required check name matches `ai-review-check-name` exactly (`AI Review Gate`).
- Ensure branch protection does not require stale/old check contexts.

## 8. Security Notes

- Never print token values in logs.
- Rotate PATs regularly.
- Use least required scopes.
- Store `ENDPOINT`, `PROJECT_TOKEN`, and `AI_REVIEW_TOKEN` as **Environment
  secrets** under a protected environment (`cliq-production`), not as plain
  repository secrets and never as `vars.*`. See section 2 for setup.
- Restrict the environment's deployment branches to protected branches only,
  so forked-repo PRs cannot trigger jobs that read these secrets.
- Review the environment's required reviewers list periodically; only admins
  or explicitly trusted maintainers should be able to approve runs or edit
  secret values.
