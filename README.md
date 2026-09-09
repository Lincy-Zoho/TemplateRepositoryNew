# GitHub Informer for Zoho Cliq — Setup Guide

---

## 1. Purpose of the workflow

The GitHub Informer workflow connects your GitHub repository to a Zoho Cliq channel. Once it is configured, important repository activity is shared in Cliq automatically, so your team does not need to keep checking GitHub manually.

It can do three things:

1. Post GitHub events to a Cliq channel. Pushes, pull requests, issues, releases, deployments, and many other events can be sent to the channel either individually or through a shared default message.
2. Keep pull request updates in one Cliq thread (optional). Instead of creating separate messages for every PR update, the workflow can continue the same conversation thread for that pull request. This requires saving the thread message ID between workflow runs, and a GitHub Project V2 field is used for that.
3. Run an AI review check on pull requests (optional). The action sends the PR diff to OpenAI, Claude, or Gemini and reports the result as a GitHub status check. If this check is marked as required, a failed review can block the merge.

The threading and AI review features are optional and independent. You can use notifications alone, notifications with threading, notifications with AI review, or all three together.

---

## 2. How to configure it

### 2.1 The channel endpoint URL

All notifications are sent to a single URL: the channel endpoint. Its format is:

```text
<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>
```

Three parts need to be filled in:

#### a. Region base

This must match the data centre where your Cliq organisation is hosted. The wrong domain fails quietly and no message is posted.

| Region | Base URL |
| --- | --- |
| US | `https://cliq.zoho.com` |
| IN | `https://cliq.zoho.in` |
| EU | `https://cliq.zoho.eu` |
| AU | `https://cliq.zoho.com.au` |
| JP | `https://cliq.zoho.jp` |

If you are unsure, check the address bar in your browser while using Cliq.

#### b. Channel unique name

This is not the display name. In Cliq, open the channel and go to Channel Actions or Info. The unique name is shown there.

Look for the value labeled **Unique Name** in the channel details panel. This is the exact value used in the URL after `/channelsbyname/`.

```text
Unique Name: githubreponotification
```

So the endpoint becomes:

```text
https://cliq.zoho.com/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx
```

The part after `/channelsbyname/` must match the channel's unique name exactly.

#### c. Webhook token

This is generated from your Zoho Cliq profile and must be created before you assemble the channel endpoint URL.

Follow these steps:

1. Click on your **profile picture** located in the top-right corner of the screen.
2. Select **Bots & Tools** from the dropdown menu.
3. Look at the left sidebar menu under the *Integrations* section and click on **Webhook Tokens**.
4. Complete the two-factor authentication (2FA) identity verification if prompted by the platform.
5. Click the **Generate New Token** button.
6. Give your token a recognizable name, then click create to view your unique hex string token.

After the token is created, copy the generated webhook token and combine it with the region base and channel unique name to build the final endpoint.

Assembled example:

```text
https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx
```

This complete string, including the `?zapikey=` part, is what you store as the `ENDPOINT` secret.

> Treat this value as a credential. Anyone who has it can post to the channel.

### 2.2 Decide: post as a user or as a bot?

Choose the mode based on who should appear as the sender of the notification:

| Mode | Who the message appears to come from | Extra setup |
| --- | --- | --- |
| User (webhook) mode | The person who created the webhook | No bot setup needed |
| Bot mode | A dedicated Cliq bot | Create the bot and add it to the channel |

Use this setting in GitHub repository variables:

- `CLIQ_NOTIFICATION_MODE=user` for user/webhook mode
- `CLIQ_NOTIFICATION_MODE=bot` for bot mode

Use user mode when you want the fastest setup and do not mind the message appearing from the webhook creator. Use bot mode when you want a stable, shared sender for ongoing notifications, especially if the person creating the webhook leaves the team or the channel.

Both modes still use the same `ENDPOINT` secret. The only extra value needed in bot mode is `CLIQ_BOT_UNIQUE_NAME`.

### 2.3 If posting as a user

In user mode, there is no extra configuration beyond the channel endpoint itself.

Use the webhook URL already assembled in section 2.1 and save it as the `ENDPOINT` secret.

```text
https://cliq.zoho.com/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx
```

No additional variable is required for user mode. `CLIQ_BOT_UNIQUE_NAME` is not used here.

### 2.4 If posting as a bot — getting the bot unique name

1. Open Zoho Cliq.
2. Click on your profile picture in the top-right corner.
3. Select **Bots & Tools**.
4. In the left sidebar, open **Integrations** and click **Bots**.
5. Select the bot you created or want to use.
6. While creating the bot, enable the required channel permissions so the bot can post messages in the target channel.
7. Open the bot details panel and look for the **API Endpoint** value.
8. The value after `/bots/` is the bot unique name.

Example:

```text
https://cliq.zoho.com/api/v2/bots/githubnotificationbot/message
```

The bot unique name is:

```text
githubnotificationbot
```

This is not necessarily the display name. It is the unique identifier that must be added in `CLIQ_BOT_UNIQUE_NAME`.

9. Add the bot to the target channel. This is the most common bot-mode failure.
10. Set the variables:

```text
CLIQ_NOTIFICATION_MODE=bot
CLIQ_BOT_UNIQUE_NAME=githubnotificationbot
```

You keep using the same `ENDPOINT` secret. The workflow appends `bot_unique_name` to the URL itself. If `CLIQ_BOT_UNIQUE_NAME` is missing while mode is `bot`, the workflow fails fast.

### 2.5 If posting PR updates into a thread

Skip this section if you are fine with each pull request event being posted as a separate message in the channel.

If you want all PR updates to continue in the same Cliq thread, set:

```text
CLIQ_THREAD_STORAGE_MODE=project
```

Why this is needed:

Each workflow run starts fresh and does not remember the previous message automatically. To continue replying in the same Cliq thread, the action must store the message ID from an earlier run and reuse it later. This is done by saving the thread ID in a custom text field on a GitHub Project V2 item.

#### Steps to create the project and custom field

1. Create or select a GitHub Project V2. Note the project number.
   - Example: `https://github.com/orgs/<org-name>/projects/1` → `1` is the `PROJECT_NUMBER`.
2. Make sure the project owner and the workflow owner are the same.
3. Add a custom field of type `Text` to the project, for example `Cliq Thread ID`.
4. Get the field identifier. Either of these values is accepted:
   - the numeric ID visible in the field settings URL.
     - Example: `https://github.com/orgs/<org-name>/projects/1/settings/fields/401236883` → `401236883` is the `PROJECT_THREAD_FIELD_ID`.
   - the GraphQL node ID starting with `PVTF_`.
5. Ensure pull requests are added to the project.
6. Create a classic PAT for `PROJECT_TOKEN` (see section 3).
7. Set the variables:

```text
CLIQ_THREAD_STORAGE_MODE=project
PROJECT_NUMBER=<project number>
PROJECT_THREAD_FIELD_ID=<field id>
```

#### How it behaves

1. The action resolves the PR item in the project.
2. It reads the thread ID from the configured field.
3. If empty, it posts a new Cliq message and captures the returned `message_id`.
4. It writes that ID back to the field.
5. Later events on that PR reuse the same thread.

If the field write fails, the action logs the error and continues. This usually means the project configuration is incorrect, not that the message was not sent. In that case, the action will still post the update as a normal Cliq channel message instead of continuing the same thread.

---

## Sample URL

```text
https://cliq.zoho.com/api/v2/channelsbyname/GitHubUpdates/message?zapikey=1001.xxxxxxx
```

This complete value is what you store as the `ENDPOINT` secret in GitHub.

---

## 3. Environment secrets — AI review token, PAT, and endpoint

**Where these go:** repository **Settings → Environments → `cliq-production` → Secrets**.

These are environment secrets, so the workflow job must declare `environment: cliq-production`; otherwise GitHub will not inject them.

| Field name | Required | Allowed values |
| --- | --- | --- |
| `ENDPOINT` | Yes | The full channel endpoint URL from section 2.1, including `?zapikey=<token>`. This is the same value for both user and bot mode. |
| `PROJECT_TOKEN` | Only if `CLIQ_THREAD_STORAGE_MODE=project` | A GitHub classic PAT with `repo` and `project` scopes. Fine-grained tokens are not supported. It must have an expiry date. |
| `AI_REVIEW_TOKEN` | Only if `AI_REVIEW_ENABLED=true` | Provider API key. OpenAI keys begin with `sk-`, Claude keys begin with `sk-ant-`, and Gemini keys begin with `AIza`. It must match the same provider as `AI_REVIEW_SERVICE` and `AI_REVIEW_MODEL`. |

### 3.1 Creating the classic PAT for `PROJECT_TOKEN`

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Set an **expiry date**. Choose a short duration and plan to rotate it before expiry. Do not create a non-expiring token.
4. Select the scopes: **`repo`** and **`project`**.
5. Copy the token and save it as the `PROJECT_TOKEN` environment secret.

> Classic PATs must be created from the **Tokens (classic)** page. Fine-grained tokens cannot write to Project V2 fields in this flow.

### 3.2 Getting the AI review token

| Provider | Where to get the key | Key prefix |
| --- | --- | --- |
| OpenAI | `platform.openai.com` → API keys | `sk-` |
| Claude | `console.anthropic.com` → API keys | `sk-ant-` |
| Gemini | Google AI Studio → Get API key | `AIza` |

The API key, endpoint, and model must all come from the same provider. For example, if you use a Claude key with the OpenAI endpoint, the request will fail with a `401` error. In practice, this often looks like a code review failure, but the real issue is usually a provider mismatch in the configuration.

If the configured AI model, service, or token is invalid, the workflow adds a clear error message as a pull request comment so the problem is easier to diagnose quickly.

#### Provider settings summary

| Provider | `AI_REVIEW_SERVICE` | `AI_REVIEW_MODEL` |
| --- | --- | --- |
| OpenAI | `openai` | `gpt-4.1-mini` |
| Claude | `claude` | `claude-sonnet-5` |
| Gemini | `gemini` | `gemini-2.5-flash` |


> Current Claude model IDs are dateless, such as `claude-opus-5`, `claude-sonnet-5`, and `claude-haiku-4-5`. Legacy aliases like `claude-3-5-sonnet-latest` should not be used for new setups.

---

## 4. Repository variables

**Where these go:** repository **Settings → Secrets and variables → Actions → Variables**.

These are repository variables, not secrets, and they are not environment-scoped. Do not hardcode them in the workflow YAML.

| Field name | Required | Allowed values |
| --- | --- | --- |
| `CLIQ_NOTIFICATION_MODE` | Yes | `user` or `bot` |
| `CLIQ_BOT_UNIQUE_NAME` | Only if mode is `bot` | The bot unique name from section 2.4, for example `githubnotificationbot`. Lower-case, no spaces. |
| `CLIQ_THREAD_STORAGE_MODE` | Yes | `project` for per-PR threads; any other value for plain channel messages. |
| `PROJECT_NUMBER` | Only if thread mode is `project` | Integer value from the project URL, for example `7`. |
| `PROJECT_THREAD_FIELD_ID` | Only if thread mode is `project` | Numeric field ID from the field settings URL, or the GraphQL node ID beginning with `PVTF_`. |
| `AI_REVIEW_ENABLED` | Yes | `true` or `false` |
| `AI_REVIEW_SERVICE` | Yes | `openai`, `claude`, or `gemini` |
| `AI_REVIEW_MODEL` | Yes | A model ID that belongs to the chosen service. See the table in section 3.2. |

The three `AI_REVIEW_*` variables are read unconditionally, so they must exist even when `AI_REVIEW_ENABLED=false`. In that case, set `AI_REVIEW_SERVICE` and `AI_REVIEW_MODEL` to any valid value.

---

## 5. Adding the workflow file

Path in the target repository: **`.github/workflows/CliqConnector.yml`**

Template repository: [https://github.com/Lincy-Zoho/TemplateRepositoryNew](https://github.com/Lincy-Zoho/TemplateRepositoryNew)

1. Open the target repository on GitHub.
2. Create the `.github/workflows` folder if it does not exist.
3. Create `CliqConnector.yml` inside it.
4. Copy the contents from the template repository.
5. Commit the file to the default branch, or to the branch you will use for the pull request, and push it.

### 5.1 Minimal workflow

```yaml
name: Communicating with Cliq

on:
  push:

jobs:
  notify:
    runs-on: ubuntu-latest
    environment: cliq-production
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.ENDPOINT }}
```

> **`environment: cliq-production` is required.** The secrets in section 3 are environment secrets, and GitHub injects them only into a job that declares the environment. Without this line, `${{ secrets.ENDPOINT }}` may appear empty, the workflow may appear to succeed, and nothing will reach Cliq. If you use a different environment name, update it here to match exactly.

### 5.2 Full workflow with AI review

```yaml
name: PR AI Review Gate

on:
  pull_request:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: read
  checks: write
  issues: write
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    environment: cliq-production
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.ENDPOINT }}
          ai-review-enabled: true
          ai-review-trigger: auto
          ai-review-on-sync: true
          ai-review-label: ai-review
          ai-review-check-name: AI Review Gate
          ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}
          ai-review-api-url: https://api.openai.com/v1/chat/completions
          ai-review-model: gpt-4.1-mini
```

The `permissions` block is required for the AI review gate. Without `checks: write` and `pull-requests: write`, the action cannot publish the status check or the review comment.

---

## 6. First run, then branch protection

Order matters here. GitHub cannot require a status check that has never been reported, so the workflow must run once before the branch rule can reference it.

1. Open or update a pull request so the workflow fires.
2. Check the **Actions** tab for the run result and confirm the message arrived in the expected Cliq channel.
3. Only after that, go to **Settings → Rules** (or **Branches → branch protection rule**).
4. Enable **Require status checks to pass**.
5. Add the check name exactly as set in `ai-review-check-name` — default is `AI Review Gate`.

Notes:

- Name matching is literal. Any mismatch leaves pull requests stuck on **Waiting for status to be reported**. After renaming the check, push a commit or re-run checks once so the new context is registered.
- Protecting all branches does not by itself make AI review block anything. The check only blocks merges where it is listed as a required status check.
- Recommended approach: protect only the release branches (`main`, `master`, or `release`) and keep AI review as PR-level validation elsewhere.
- In practice, add the pre-live branches that should gate deployments before code moves forward, such as `main`, `master`, and `release`, to the branch protection rule so the workflow runs and enforces the check before merge.

---

## 7. Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Workflow succeeds, nothing in Cliq | Job missing `environment: cliq-production`, so `ENDPOINT` is empty |
| Message never appears, no error | Wrong region base domain (2.1) |
| Nothing posted in bot mode | Bot not added to the channel (2.4, step 3) |
| Workflow fails immediately in bot mode | `CLIQ_BOT_UNIQUE_NAME` not set |
| Every PR event is a new message instead of a thread reply | Project field write failing — check PAT type and scopes, that the field is a Text field, and that the PR is actually an item in the project |
| PR stuck on *Waiting for status to be reported* | Required check name does not match `ai-review-check-name` |
| Cannot find the check to require | Workflow has not run yet (section 6) |
| AI review returns `401` or shows a failed review comment | Token, endpoint, and model from different providers, or invalid model/service/token configuration; the action also posts the error in the PR comment for diagnosis |
| `${{ vars.* }}` resolves empty | Value created as a secret, or under the environment instead of Actions → Variables |

### Quick checks

1. Confirm the workflow job declares the correct environment.
2. Confirm the `ENDPOINT` secret is the complete channel URL including the `?zapikey=` value.
3. Confirm the channel unique name and region base match the actual Cliq channel.
4. Confirm `CLIQ_NOTIFICATION_MODE` is set correctly to `user` or `bot`.
5. If bot mode is enabled, confirm the bot is a member of the target channel and `CLIQ_BOT_UNIQUE_NAME` is set exactly.
6. If thread mode is enabled, confirm the project field is a `Text` field, the PAT is a classic PAT with `repo` and `project` scopes, and the PR is included in the project.
7. Confirm the check name in the branch protection rule matches the value used in `ai-review-check-name` exactly.
8. Confirm the AI provider values all come from the same service and the token matches the provider configuration.

### Final setup checklist

Use this checklist before creating the workflow and again before enabling branch protection:

- [ ] Cliq channel exists and the endpoint URL is valid.
- [ ] `ENDPOINT` is saved as an environment secret under `cliq-production`.
- [ ] `CLIQ_NOTIFICATION_MODE` is set to `user` or `bot`.
- [ ] If bot mode is used, the bot is added to the channel and `CLIQ_BOT_UNIQUE_NAME` is correct.
- [ ] If thread mode is enabled, the project exists, the PR is included in the project, and the thread ID field is a `Text` field.
- [ ] `PROJECT_TOKEN` is a classic PAT with `repo` and `project` scope.
- [ ] Repository variables are created under **Settings → Secrets and variables → Actions → Variables**.
- [ ] `AI_REVIEW_ENABLED`, `AI_REVIEW_SERVICE`, and `AI_REVIEW_MODEL` are set correctly.
- [ ] `AI_REVIEW_TOKEN` matches the selected provider and model.
- [ ] The workflow file is committed and the first run succeeds.
- [ ] Branch protection is configured for the required pre-live branches like `main`, `master`, and `release`.
- [ ] The required check name matches `ai-review-check-name` exactly.

### Example branch filters

If you want the workflow to run only on the main release paths, use:

```yaml
on:
  pull_request:
    branches: [main, master, release]
  push:
    branches: [main, master, release]
```

This helps ensure the workflow only runs where your deployment and merge rules are enforced.

---


