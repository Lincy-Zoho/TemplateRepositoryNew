# GitHub Informer for Zoho Cliq
The GitHub Action is used to integrate GitHub and Zoho Cliq, by notifying about the GitHub Events performed, to the Zoho Cliq Channels.

GitHub Informer requires the following inputs to integrate the **GitHub Actions** with your **Cliq** channels
- Cliq Webhook Token
- Cliq Channel API Endpoint
- Individual messages for each of the GitHub events (in the name of **event**-message)
- A default message that you want to send if the message is not specified for that event.

Two posting modes are supported:

1. Webhook mode
  - **channel-endpoint**: `<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>`

2. Bot-auth mode
  - **channel-endpoint**: `https://cliq.zoho.com/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?bot_unique_name=<BOT_UNIQUE_NAME>`
  - **channel-auth-token**: Cliq OAuth bearer token
  - the **bot must already be created** by the user and **added to the target channel**

### Where to set bot or user notification mode

Set this in your workflow file: `.github/workflows/CliqConnector.yml`.

You can provide these values in **either** of these two ways:

1. **GitHub Variables** (recommended)
2. **Directly in the workflow YAML**

Use the job-level `env` keys:

```yaml
env:
  CLIQ_NOTIFICATION_MODE: ${{ vars.CLIQ_NOTIFICATION_MODE || 'bot' }}
  CLIQ_BOT_UNIQUE_NAME: ${{ vars.CLIQ_BOT_UNIQUE_NAME || 'githubnotification' }}
```

- Set `CLIQ_NOTIFICATION_MODE` to `user` for normal webhook/user mode.
- Set `CLIQ_NOTIFICATION_MODE` to `bot` for bot notification mode.
- **`CLIQ_BOT_UNIQUE_NAME` is required when mode is `bot`.**

Option 1: GitHub Variables (recommended)

1. In GitHub repository settings, go to `Secrets and variables` -> `Actions` -> `Variables`.
2. Create **`CLIQ_NOTIFICATION_MODE`** with value **`user`** or **`bot`**.
3. If mode is `bot`, create **`CLIQ_BOT_UNIQUE_NAME`** with your actual bot unique name.

Option 2: Directly in `CliqConnector.yml`

```yaml
env:
  CLIQ_NOTIFICATION_MODE: bot
  CLIQ_BOT_UNIQUE_NAME: githubnotification
```

Use this if you want fixed values in the workflow file itself.

### Bot mode setup

If you want **bot notification mode**, configure:

1. **Create the bot in Zoho Cliq**.
2. **Get the bot unique name** from the bot configuration/details page.
  - Open **Integrations** -> **Bots**.
  - Select your bot.
  - In the bot details panel, check the **API Endpoint**.
  - The value after `/bots/` in the endpoint is the **bot unique name**.
  - Example: if the API endpoint is `https://cliq.zoho.com/api/v2/bots/githubnotification/message`, then the bot unique name is **`githubnotification`**.
3. **Add the bot to the target Cliq channel** where notifications should be posted.
4. In GitHub repository settings, go to `Secrets and variables` -> `Actions` -> `Variables`.
5. Create **`CLIQ_NOTIFICATION_MODE=bot`**.
6. Create **`CLIQ_BOT_UNIQUE_NAME=<your bot unique name>`**.

In bot mode:

- The workflow reuses the existing **`ENDPOINT`** secret.
- The workflow appends **`bot_unique_name`** automatically.
- If **`CLIQ_BOT_UNIQUE_NAME`** is missing, workflow fails fast.

### User mode setup

If you want normal user/webhook notification mode, configure:

1. `CLIQ_NOTIFICATION_MODE=user`

In user mode:

- **`CLIQ_BOT_UNIQUE_NAME` is not required.**
- Workflow will use the webhook endpoint as-is.
- **You do not need to reconfigure endpoint in this section**; keep using the existing **`ENDPOINT`** secret configured under **GitHub Secret for Channel Endpoint**.

## Available Regions

Use the same base domain as your Zoho Cliq organization region when configuring the channel endpoint.

Common region domains:

- US: `https://cliq.zoho.com`
- IN: `https://cliq.zoho.in`
- EU: `https://cliq.zoho.eu`
- AU: `https://cliq.zoho.com.au`
- JP: `https://cliq.zoho.jp`

Endpoint pattern:

- `<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>`

Examples:

- `https://cliq.zoho.com/api/v2/channelsbyname/GitHubupdates/message?zapikey=1001.xxxxx`
- `https://cliq.zoho.in/api/v2/channelsbyname/GitHubupdates/message?zapikey=1001.xxxxx`
  
## GitHub Secret for Channel Endpoint 🔗
You must add **GitHub Secret** which contains the channel endpoint in the format 

```
<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>
```

You must create a **GitHub Secret** for providing the **Channel Endpoint** by
  - Go to the Repository where the CliqInformer will be added and go to the '**Settings**' tab.
  - Select '**Secrets and variables**' and click on '**Actions**' in the dropdown.
  - Click on '**New repository secret**' and enter the name of your secret and also enter the Cliq channel endpoint  as the Secret (in above mentioned format)

and use the secret as the '**channel-endpoint**' input in the job of your workflow.

```yaml
  steps:
    - uses: Integrations-dev/GitHub-Informer@v1
      with:
        channel-endpoint: ${{ secrets.SECRET_NAME }}
```

## Store Cliq Thread ID in GitHub Project

For PR events, you can store and reuse the Cliq thread ID in a GitHub Project V2 custom text field (recommended).

Create GitHub classic token for `PROJECT_TOKEN`:

1. Open https://github.com/settings/tokens
2. Click Generate new token (classic)
3. Select scopes:
  - **`repo`**
  - **`project`**
4. Copy the token and save it as repository secret **`PROJECT_TOKEN`**

Required configuration:

- Secret: **`PROJECT_TOKEN`**
  - Use a classic PAT with **`repo`** and **`project`** scopes.
- Env: **`CLIQ_THREAD_STORAGE_MODE=project`**
- Env: **`GITHUB_PROJECT_OWNER=<owner login>`**
- Env: **`GITHUB_PROJECT_NUMBER=<project number>`**
- Env: **`GITHUB_PROJECT_THREAD_FIELD_ID=<field identifier>`**
  - Can be either:
    - numeric field identifier from the Project field settings URL, or
    - GraphQL field node id (`PVTF_*`).

Behavior:

1. Action resolves the PR item in the configured Project V2.
2. Action reads existing thread ID from the configured custom field.
3. If not found, action posts a new Cliq message and captures `message_id`.
4. Action writes captured thread ID to the same project field.
5. Later PR events reuse that saved value and continue in the same Cliq thread.

If project mode is enabled and project field update fails, action logs the failure. Ensure **`PROJECT_TOKEN`**, **project owner/number**, and **field identifier** are correct.

## Custom Event Messages ⚙️

Suppose you need a notification in Cliq for only selected events or actions,
  - you may change the '**_on_**' key of the YAML File where the Action is called.
  - Or additionally, you can set all the required messages and declare the input '**set-message-if-none**' as '**false**' to avoid messages from events you don't want. 
  
You provide default messages to all kind of actions that triggers a workflow.

The messages can be customized by giving the input '**_event_-message**' (where '_event_' is the event for which you would like to customize the message).

For ex: To set a custom message for a Pull Request event, you must define the input as,

```yaml
  pull-request-message: 'A Pull Request has been Opened'
```

## Default Message 📓

If you wish to add a single custom message for all kinds of events, you may use the '**default-message**'. 

For ex: To set a default custom message , you must define the _default-message_ as

```yaml
  default-message: 'A (event) has been (action)'
```

## Shortcuts ⏩

We also provide several shortcuts to obtain the variables that you want to insert in the message, such as,
  - **(event)**: which will be replaced with the event that the workflow is triggered by
  - **(action)**: which will be replaced by the Action the Event is performing with
  - **(me)**: which will be replaced with the GitHub user performing the action.
  - **(repo)**: which will be replaced by the Repository where the GitHub action is performed
  - **(ref)**: which will be replaced by the Branch/Tag where the GitHub action is performed
  - **(workflow)**: which will be replaced by the workflow on which the GitHub Action is performed
  - **(rule)**: which will be replaced by the Branch Protection Rule (if the Event is Branch Protection Rule)
  - **(run)**: which will be replaced by the Check Run (if the Event is Check Run)
  - **(branch)**: which will be replaced by the Branch/Tag which is Created/Deleted (if the Event is Create / Delete)
  - **(deployment)**: which will be replaced by the deployment (if the Event is Deployment or Deployment Status)
  - **(discussion)**: which will be replaced by the discussion which is worked on
  - **(category)**: which will be replaced by the Category Name to which the discussion is changed to
  - **(issue)**: which will be replaced by the issue (if the Event is Issue or Issue Comment)
  - **(label)**: which will be replaced by the label that is being worked on or added to
  - **(milestone)**: which will be replaced by the milestone that is being worked on or added to
  - **(assignee)**: which will be replaced by the User Assigned to the Issue or Pull Request
  - **(pull)**: which will be replaced by the Pull Request that is worked on
  - **(package)**: which will be replaced the Registry Package that is being worked on
  - **(release)**: which will be replaced by the release that is being worked on
  - **(status)**: which will be replaced by the Status of the Event (if the Event is Deployment Status or Status)

Example:

A GitHub Action is triggered by (me) at (repo).

will change to 

A GitHub Action is triggered by [user_name](https://www.github.com/user_name) at [user_name/repository_name](https://www.github.com/user_name/repository_name).

Upon successfully providing the inputs as per criteria, the message will be successfully sent to the Cliq Channel.

The GitHub events that trigger a workflow are listed below, among which all events are supported by GitHub Informer

|    branch_protection_rule    |          check_run          |          check_suite         |            create            |           delete            |
|            :----:            |           :----:            |            :----:            |            :----:            |           :----:            |
| **deployment**               | **deployment_status**       | **discussion**               | **discussion_comment**       | **fork**                    |
| **gollum**                   | **issue_comment**           | **issues**                   | **label**                    | **milestone**               |
| **page_build**               | **public**                  | **pull_request**             | **pull_request_comment**     | **pull_request_review**     |
|**pull_request_review_comment**| **pull_request_target**    | **push**                     | **registry_package**         | **release**                 |
| **repository_dispatch**     | **schedule**                 | **status**                   | **watch**                    | **workflow_dispatch**       |

## Base YAML Code 🗒

Don't worry about remembering a lot of stuff. Here is the minimal code that's required to start with. 

```yaml
name : Communicating with Cliq
on:
  #you may add the events you like to get notified
  push:
    
jobs:
  test_name:
    runs-on: ubuntu-latest
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.ENDPOINT }}
```

That's all! You will start getting notified for each event occurring in GitHub through the GitHub Action.

Go to the Actions tab of the repository to view the message status.

## AI Review Gate (OpenAI, Claude, Gemini)

You can enable an AI review gate for pull requests. The action can run against OpenAI, Claude, or Gemini.

- If the AI decision is pass, the check passes and no PR comment is added.
- If the AI decision is fail (or the AI call fails), the check fails, a PR comment is added, and a failure message is posted to the Cliq PR thread.

### Required GitHub Workflow Permissions

Your workflow must include:

```yaml
permissions:
  contents: read
  checks: write
  issues: write
  pull-requests: write
```

### Recommended Workflow Usage

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
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.CLIQ_ENDPOINT }}
          ai-review-enabled: true
          ai-review-trigger: auto
          ai-review-on-sync: true
          ai-review-label: ai-review
          ai-review-check-name: AI Review Gate
          ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}
          ai-review-api-url: https://api.openai.com/v1/chat/completions
          ai-review-model: gpt-4.1-mini
```

### Provider Configuration

Only the token should be stored as a secret. Endpoint and model can be plain workflow values.

OpenAI

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with sk-
ai-review-api-url: https://api.openai.com/v1/chat/completions
ai-review-model: gpt-4.1-mini
```

Claude

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with sk-ant-
ai-review-api-url: https://api.anthropic.com/v1/messages
ai-review-model: claude-3-5-sonnet-latest
```

Gemini

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with AIza
ai-review-api-url: https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent
ai-review-model: gemini-1.5-pro
```

### Branch Protection

To enforce mentor approval policy, add the check name from **`ai-review-check-name`** (default: **`AI Review Gate`**) as a **required status check** in branch protection.

### Configure Required Status Check (Step-by-step)

1. In workflow, set the check name you want:

```yaml
ai-review-check-name: AI Review Gate
```

2. Run the workflow once on a PR so GitHub can see this check context.

3. In GitHub repository settings, open `Rules` (or `Branches` -> branch protection rule).

4. Enable `Require status checks to pass`.

5. Add the same check name exactly (example: `AI Review Gate`) under required checks.

Important:

- **If you rename `ai-review-check-name`, update the required check name in rules to the same value.**
- **Name matching is exact.** Any mismatch causes **`Waiting for status to be reported`**.
- After changing rules/check name, **push a new commit or re-run checks once** so the new context is picked.

Here is a Template Repository for the GitHub Informer which you can use as a baseline to work with and customize to your usage.
https://www.github.com/Integrations-dev/RepositoryTemplate
